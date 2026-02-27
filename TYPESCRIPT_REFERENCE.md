# TypeScript Reference Implementation

This document provides a TypeScript reference implementation of the
`io.modelcontextprotocol/content-negotiation` extension using the
[official MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk).

## Overview

The implementation has two layers:

1. **`Features`** — a class with convenience properties for the most common
   negotiation axes, mirroring the Python `Features` dataclass.
2. **`getFeatures(mcp)`** — reads client capabilities from the `McpServer`
   instance and returns a `Features` object. No middleware required; the
   high-level `McpServer` exposes its underlying `Server` via `mcpServer.server`,
   which provides `getClientCapabilities()` after the `oninitialized` callback
   fires.

## Implementation

```typescript
// content-negotiation.ts

import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

const EXTENSION_ID = "io.modelcontextprotocol/content-negotiation";

export class Features {
  constructor(private readonly tags: string[] = []) {}

  // --- Convenience properties ---

  get isAgent(): boolean {
    return this.tags.includes("agent");
  }

  get isHuman(): boolean {
    return this.tags.includes("human");
  }

  get isInteractive(): boolean {
    return this.tags.includes("interactive") && !this.tags.includes("!interactive");
  }

  get hasSampling(): boolean {
    return this.tags.includes("sampling");
  }

  get hasElicitation(): boolean {
    return this.tags.includes("elicitation");
  }

  get format(): "json" | "text" | "markdown" {
    for (const tag of this.tags) {
      if (tag.startsWith("format=")) {
        const value = tag.slice("format=".length);
        if (value === "json" || value === "text" || value === "markdown") {
          return value;
        }
      }
    }
    return "markdown"; // default
  }

  get verbosity(): "compact" | "standard" | "verbose" {
    for (const tag of this.tags) {
      if (tag.startsWith("verbosity=")) {
        const value = tag.slice("verbosity=".length);
        if (value === "compact" || value === "standard" || value === "verbose") {
          return value;
        }
      }
    }
    return "standard"; // default
  }

  /** Check for any feature tag, including vendor tags (x-*). */
  hasTag(tag: string): boolean {
    return this.tags.includes(tag);
  }

  /** Return the value of a key=value vendor tag, or undefined. */
  vendorValue(prefix: string): string | undefined {
    for (const tag of this.tags) {
      if (tag.startsWith(`${prefix}=`)) {
        return tag.slice(prefix.length + 1);
      }
    }
    return undefined;
  }
}

/**
 * Return the negotiated Features for the current session.
 *
 * Falls back to an empty Features (all defaults) when no negotiation
 * took place or the extension was not declared by the client.
 *
 * Call this at the top of any tool handler. It is safe to call on every
 * request — `getClientCapabilities()` is a simple in-memory lookup.
 */
export function getFeatures(mcp: McpServer): Features {
  const capabilities = mcp.server.getClientCapabilities();
  const tags: string[] =
    (capabilities?.extensions?.[EXTENSION_ID] as { features?: string[] })
      ?.features ?? [];
  return new Features(tags);
}
```

## Server Example

```typescript
// server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import { getFeatures } from "./content-negotiation.js";

const mcp = new McpServer(
  { name: "Weather Service", version: "1.0.0" },
  {
    capabilities: {
      // Advertise support for the content negotiation extension.
      // The empty object signals availability; feature tag parsing
      // is done on the client capabilities received at initialize.
      extensions: {
        "io.modelcontextprotocol/content-negotiation": {},
      },
    },
  },
);

mcp.tool(
  "get_weather",
  "Return weather data for a location.",
  { location: z.string().describe("City or region name") },
  async ({ location }) => {
    const features = getFeatures(mcp);

    // Raw data — always computed the same way regardless of negotiation
    const data = {
      location,
      temperature_c: 8,
      humidity_percent: 72,
      precipitation_probability: 0.3,
      wind_speed_kmh: 15,
      uv_index: 2,
    };

    if (features.isAgent && features.format === "json") {
      // Compact structured payload for agents
      if (features.verbosity === "compact") {
        return { content: [{ type: "text", text: JSON.stringify(data) }] };
      }
      // Verbose: add units and metadata
      const verbose = {
        ...data,
        units: "metric",
        source: "WeatherAPI",
        valid_for_minutes: 60,
      };
      return { content: [{ type: "text", text: JSON.stringify(verbose) }] };
    }

    // Human-readable markdown
    const lines = [
      `## Weather in ${location}`,
      `- **Temperature**: ${data.temperature_c}°C`,
      `- **Humidity**: ${data.humidity_percent}%`,
      `- **Precipitation**: ${(data.precipitation_probability * 100).toFixed(0)}% chance`,
      `- **Wind**: ${data.wind_speed_kmh} km/h`,
    ];
    if (features.verbosity === "verbose") {
      lines.push(
        `- **UV Index**: ${data.uv_index} (low)`,
        "",
        "_Data provided by WeatherAPI. Valid for 60 minutes._",
      );
    }
    return { content: [{ type: "text", text: lines.join("\n") }] };
  },
);

const transport = new StdioServerTransport();
await mcp.connect(transport);
```

## Client Initialization Example

```typescript
// client-example.ts — declares content negotiation at initialize
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

const transport = new StdioClientTransport({
  command: "node",
  args: ["server.js"],
});

const client = new Client(
  { name: "my-agent", version: "1.0.0" },
  {
    capabilities: {
      extensions: {
        "io.modelcontextprotocol/content-negotiation": {
          version: "1.0",
          features: [
            "agent",
            "sampling",
            "format=json",
            "verbosity=compact",
          ],
        },
      },
    },
  },
);

await client.connect(transport);

const result = await client.callTool({
  name: "get_weather",
  arguments: { location: "Bern" },
});
console.log(result); // → compact JSON string
```

## Design Notes

- **No middleware needed**: Unlike the Python/FastMCP implementation, the
  TypeScript SDK exposes `mcpServer.server.getClientCapabilities()` directly.
  There is no per-request parsing — the SDK caches capabilities after
  `initialize`, so `getFeatures()` is a pure in-memory lookup.
- **Session-scoped**: `getClientCapabilities()` returns the capabilities
  captured during the `initialize` handshake; they are the same for the
  lifetime of the connection, matching the session-scoped negotiation model.
- **Safe defaults**: `Features` properties all have sensible defaults
  (`format="markdown"`, `verbosity="standard"`) so tools degrade gracefully
  when no negotiation occurred.
- **Security**: feature tags only influence content shape — the tool always
  computes the same underlying data and only varies the presentation.
- **Extensible**: `Features.hasTag()` and `Features.vendorValue()` support
  vendor tags (`x-mycompany-hint=value`) without any changes to `getFeatures`.
- **Capability advertisement**: passing the extension key in the `McpServer`
  constructor's `capabilities.extensions` object signals to clients that the
  server understands and honours the extension.
