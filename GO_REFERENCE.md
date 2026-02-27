# Go Reference Implementation

This document provides a Go reference implementation of the
`io.modelcontextprotocol/content-negotiation` extension using the
[official MCP Go SDK](https://github.com/modelcontextprotocol/go-sdk).

## Overview

The implementation has two layers:

1. **`Features`** — a struct with convenience methods for the most common
   negotiation axes, mirroring the Python `Features` Pydantic model and
   TypeScript `Features` class.
2. **`GetFeatures(req)`** — reads client capabilities from the
   `*mcp.CallToolRequest` passed to every tool handler and returns a `Features`
   value. No middleware required; the SDK exposes the session's `InitializeParams`
   directly on the request object.

## Dependencies

```go
// go.mod
require github.com/modelcontextprotocol/go-sdk v0.2.0
```

```go
import "github.com/modelcontextprotocol/go-sdk/mcp"
```

## Implementation

```go
// content_negotiation.go
package main

import (
	"strings"

	"github.com/modelcontextprotocol/go-sdk/mcp"
)

const extensionID = "io.modelcontextprotocol/content-negotiation"

// Features holds the negotiated content preferences for the current MCP session.
// All methods return safe defaults when no negotiation took place.
type Features struct {
	tags []string
}

// Empty is returned when no negotiation took place.
var Empty = Features{}

// IsAgent reports whether the client declared the "agent" tag.
func (f Features) IsAgent() bool { return f.hasTag("agent") }

// IsHuman reports whether the client declared the "human" tag.
func (f Features) IsHuman() bool { return f.hasTag("human") }

// IsInteractive reports whether "interactive" is present and "!interactive" is absent.
func (f Features) IsInteractive() bool {
	return f.hasTag("interactive") && !f.hasTag("!interactive")
}

// HasSampling reports whether the client declared the "sampling" tag.
func (f Features) HasSampling() bool { return f.hasTag("sampling") }

// HasElicitation reports whether the client declared the "elicitation" tag.
func (f Features) HasElicitation() bool { return f.hasTag("elicitation") }

// Format returns the negotiated format: "json", "text", or "markdown" (default).
func (f Features) Format() string {
	for _, t := range f.tags {
		if strings.HasPrefix(t, "format=") {
			v := t[len("format="):]
			if v == "json" || v == "text" || v == "markdown" {
				return v
			}
		}
	}
	return "markdown"
}

// Verbosity returns the negotiated verbosity: "compact", "standard" (default), or "verbose".
func (f Features) Verbosity() string {
	for _, t := range f.tags {
		if strings.HasPrefix(t, "verbosity=") {
			v := t[len("verbosity="):]
			if v == "compact" || v == "standard" || v == "verbose" {
				return v
			}
		}
	}
	return "standard"
}

// HasTag checks for any feature tag, including vendor tags (x-*).
func (f Features) HasTag(tag string) bool { return f.hasTag(tag) }

// VendorValue returns the value of a "prefix=value" vendor tag, and whether it was found.
//
// Example: f.VendorValue("x-mycompany-hint") returns ("value", true) when the
// tag "x-mycompany-hint=value" is present.
func (f Features) VendorValue(prefix string) (string, bool) {
	for _, t := range f.tags {
		if strings.HasPrefix(t, prefix+"=") {
			return t[len(prefix)+1:], true
		}
	}
	return "", false
}

func (f Features) hasTag(tag string) bool {
	for _, t := range f.tags {
		if t == tag {
			return true
		}
	}
	return false
}

// GetFeatures returns the negotiated Features for the current session.
//
// Falls back to Empty (all defaults) when no negotiation took place or the
// extension was not declared by the client.
//
// Call this at the top of any tool handler. It is safe to call on every
// request — InitializeParams is a simple in-memory lookup on the session.
func GetFeatures(req *mcp.CallToolRequest) Features {
	session, ok := req.GetSession().(*mcp.ServerSession)
	if !ok || session == nil {
		return Empty
	}
	params := session.InitializeParams()
	if params == nil || params.Capabilities == nil {
		return Empty
	}
	ext, ok := params.Capabilities.Extensions[extensionID]
	if !ok {
		return Empty
	}
	extMap, ok := ext.(map[string]any)
	if !ok {
		return Empty
	}
	rawList, ok := extMap["features"].([]any)
	if !ok {
		return Empty
	}
	tags := make([]string, 0, len(rawList))
	for _, v := range rawList {
		if s, ok := v.(string); ok {
			tags = append(tags, s)
		}
	}
	return Features{tags: tags}
}
```

## Server Example

```go
// main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"strings"

	"github.com/modelcontextprotocol/go-sdk/mcp"
)

type WeatherArgs struct {
	Location string `json:"location" jsonschema:"City or region name"`
}

func main() {
	server := mcp.NewServer(
		&mcp.Implementation{Name: "Weather Service", Version: "1.0.0"},
		nil,
	)

	// Advertise support for the content negotiation extension.
	// The empty map signals availability; feature tag parsing is done on
	// the client capabilities received at initialize.
	server.AddCapabilityExtension(
		"io.modelcontextprotocol/content-negotiation",
		map[string]any{},
	)

	mcp.AddTool(server, &mcp.Tool{
		Name:        "get_weather",
		Description: "Return weather data for a location.",
	}, func(ctx context.Context, req *mcp.CallToolRequest, args WeatherArgs) (
		*mcp.CallToolResult, any, error,
	) {
		features := GetFeatures(req)

		// Raw data — always computed the same way regardless of negotiation
		data := map[string]any{
			"location":                  args.Location,
			"temperature_c":             8,
			"humidity_percent":          72,
			"precipitation_probability": 0.30,
			"wind_speed_kmh":            15,
			"uv_index":                  2,
		}

		var text string
		if features.IsAgent() && features.Format() == "json" {
			if features.Verbosity() == "compact" {
				// Compact structured payload for agents
				b, _ := json.Marshal(data)
				text = string(b)
			} else {
				// Verbose: add units and metadata
				verbose := make(map[string]any, len(data)+3)
				for k, v := range data {
					verbose[k] = v
				}
				verbose["units"] = "metric"
				verbose["source"] = "WeatherAPI"
				verbose["valid_for_minutes"] = 60
				b, _ := json.Marshal(verbose)
				text = string(b)
			}
		} else {
			// Human-readable markdown
			lines := []string{
				fmt.Sprintf("## Weather in %s", args.Location),
				fmt.Sprintf("- **Temperature**: %v°C", data["temperature_c"]),
				fmt.Sprintf("- **Humidity**: %v%%", data["humidity_percent"]),
				fmt.Sprintf("- **Precipitation**: %d%% chance", int(0.30*100)),
				fmt.Sprintf("- **Wind**: %v km/h", data["wind_speed_kmh"]),
			}
			if features.Verbosity() == "verbose" {
				lines = append(lines,
					fmt.Sprintf("- **UV Index**: %v (low)", data["uv_index"]),
					"",
					"_Data provided by WeatherAPI. Valid for 60 minutes._",
				)
			}
			text = strings.Join(lines, "\n")
		}

		return &mcp.CallToolResult{
			Content: []mcp.Content{
				&mcp.TextContent{Text: text},
			},
		}, nil, nil
	})

	if err := server.Run(context.Background(), &mcp.StdioTransport{}); err != nil {
		log.Fatal(err)
	}
}
```

## Client Initialization Example

```go
// client_example.go — declares content negotiation at initialize
package main

import (
	"context"
	"log"
	"os/exec"

	"github.com/modelcontextprotocol/go-sdk/mcp"
)

func main() {
	ctx := context.Background()

	caps := &mcp.ClientCapabilities{}
	caps.AddExtension("io.modelcontextprotocol/content-negotiation", map[string]any{
		"version": "1.0",
		"features": []string{
			"agent",
			"sampling",
			"format=json",
			"verbosity=compact",
		},
	})

	client := mcp.NewClient(
		&mcp.Implementation{Name: "my-agent", Version: "1.0.0"},
		&mcp.ClientOptions{Capabilities: caps},
	)

	transport := &mcp.CommandTransport{
		Command: exec.Command("./weather-server"),
	}
	session, err := client.Connect(ctx, transport, nil)
	if err != nil {
		log.Fatal(err)
	}
	defer session.Close()

	result, err := session.CallTool(ctx, &mcp.CallToolParams{
		Name:      "get_weather",
		Arguments: map[string]any{"location": "Bern"},
	})
	if err != nil {
		log.Fatal(err)
	}
	log.Println(result) // → compact JSON string
}
```

## Design Notes

- **No middleware needed**: The Go SDK passes `*mcp.CallToolRequest` to every
  tool handler. Calling `req.GetSession().(*mcp.ServerSession).InitializeParams()`
  gives direct access to the capabilities captured at `initialize` — no
  middleware, no context values.
- **Session-scoped**: `InitializeParams()` returns the data from the `initialize`
  handshake; it is the same for the lifetime of the connection, matching the
  session-scoped negotiation model.
- **Safe type assertion chain**: `ClientCapabilities.Extensions` is a
  `map[string]any`. Each step in `GetFeatures` uses the two-value `ok` form of
  type assertion; any unexpected shape falls back to `Empty` without panicking.
- **Value receiver on `Features`**: `Features` is a small struct (a slice header);
  value receivers are idiomatic Go and avoid accidental mutation.
- **Safe defaults**: all `Features` methods return sensible defaults
  (`Format()` → `"markdown"`, `Verbosity()` → `"standard"`) so tools degrade
  gracefully when no negotiation occurred.
- **Security**: feature tags only influence content shape — the tool always
  computes the same underlying data and only varies the presentation. Tags never
  control authentication or access decisions.
- **Extensible**: `Features.HasTag()` and `Features.VendorValue()` support vendor
  tags (`x-mycompany-hint=value`) without any changes to `GetFeatures`.
