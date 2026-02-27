# SEP-XXXX: Content Negotiation Extension

- **Status**: Draft
- **Version**: v0.9.2
- **Type**: Extensions Track
- **Extension ID**: `io.modelcontextprotocol/content-negotiation`
- **Created**: 2026-02-22
- **Author(s)**:
  - Aaron Sempf <sempfa@amazon.com>
  - Andreas Schlapbach <andreas.schlapbach@sbb.ch>
- **Sponsor**: (Seeking)
- **PR**: (To be filled)
- **Extension Repository**: (To be created at
  `https://github.com/modelcontextprotocol/ext-content-negotiation`)

## Abstract

This extension introduces **content negotiation** to MCP, enabling clients and
servers to negotiate response format based on client capabilities and
preferences.

The core problem: MCP servers cannot distinguish between AI agent clients and
human-facing clients, leading to one-size-fits-all content that serves neither
audience well. This extension proposes a **transparent content negotiation**
mechanism inspired by RFC 2295, where clients declare their capabilities during
the `initialize` handshake, and servers respond with structurally different
content variants optimized for those capabilities.

The key insight is that negotiation axes are **the client's declared MCP
capabilities** (sampling, elicitation, roots, tasks, etc.), not generic "agent"
vs "human" labels. A server knowing that a client has `sampling` but no
`elicitation` should produce different output than when the client has
both—because the client's actual protocol capabilities determine what responses
are useful.

This extension is **optional** and **fully backward compatible**: servers
without it behave identically to today's protocol; clients without it work
unchanged with any server.

## About This Extension

This SEP proposes an official MCP extension following the framework defined in
[SEP-2133: Extensions](https://modelcontextprotocol.io/community/seps/2133-extensions).

- **Extension ID**: `io.modelcontextprotocol/content-negotiation`
- **Scope**: Adds optional `contentNegotiation` capability to
  `ClientCapabilities` and `ServerCapabilities`
- **Impact**: Allows clients to declare content preferences; servers may vary
  response format accordingly
- **Adoption**: Optional; clients and servers may ignore this extension entirely
  with no loss of conformance
- **Versioning**: Independently versioned from core protocol; may evolve without
  core protocol changes

## Motivation

### The Problem: One Format Cannot Serve Two Audiences

Consider a weather service integrated with MCP. Currently, it returns the same
response format regardless of whether the client is:

- An AI agent that will embed the result in a tool call and continue reasoning
- A human in a chat interface who will read the markdown and decide next steps

**Current Approach (Agent receives)**:

```
Current temperature in Bern: 8°C. Humidity is 72%. There is a 30% chance
of precipitation in the next 2 hours. The forecast shows gradually warming
trends over the coming week, with temperatures reaching 12°C by Thursday.
UV index is 2 (low). Wind speed is 15 km/h from the northwest. This is
typical February weather for the region...
```

**What the agent actually needs**:

```json
{
  "location": "Bern",
  "temperature_c": 8,
  "humidity_percent": 72,
  "precipitation_probability": 0.3,
  "wind_speed_kmh": 15,
  "uv_index": 2
}
```

**What the human needs** (what we currently send):

- Rich markdown with context
- Narrative structure explaining implications
- Links to related resources
- Natural language phrasing

### Why Current Workarounds Fail

#### 1. **Duplicate Tools**

Some servers expose two versions of every tool:

- `weather_current_prose` (for humans)
- `weather_current_json` (for agents)

**Problems**:

- 2x maintenance burden
- Discovery is ambiguous ("which should I use?")
- Scales poorly (10 tools → 20 tools)
- LLM confusion over redundant options

#### 2. **Heuristic Detection via `clientInfo.name`**

Servers try to guess: "Is it Claude? Is it ChatGPT? Is it an agent?"

**Problems**:

- Unreliable (custom clients have arbitrary names)
- Not future-proof (new client types constantly appearing)
- Security risk (can be spoofed)
- Doesn't reflect actual protocol capabilities

#### 3. **`annotations.audience` Per-Block**

MCP's existing `annotations` feature allows marking blocks with audience hints.
But it's:

- **Per-message, not session-scoped**: Server doesn't know upfront what to send
- **Not negotiated**: Server can't promise a particular format
- **Coarse-grained**: Can't say "I need JSON for this tool result AND rich
  markdown for this prompt"

#### 4. **`experimental` Capability for Custom Behavior**

Some servers use the `experimental` capability to signal non-standard features.

**Problems**:

- No standardized semantics: What does it mean?
- No registry: How does a client declare support?
- No interoperability: Each server invents its own dialect
- Pollutes the capability namespace

### The Core Issue: Lack of Negotiation

All workarounds share a root problem: **servers cannot negotiate content with
clients upfront**. They must either:

- Guess (unreliable)
- Support all formats simultaneously (bloated)
- Require human configuration (not scalable)

### Real-World Use Cases Failing Today

#### 1. **Journey Service (Public Transport)**

- **Problem**: Tool results are prose descriptions of connections. An AI agent
  needs structured data (departure times, platform numbers, vehicle types).
- **Current**: Two separate endpoints, confusing the LLM
- **Needed**: One tool, content shape varies by client capability

#### 2. **Geospatial/Mapping Service**

- **Problem**: Geographic data comes as rich narratives (landmarks, directions,
  history). An agent building a route needs coordinates, boundaries, and feature
  types. A human needs readable descriptions.
- **Current**: Agents strip formatting, waste context; humans can't parse raw
  GeoJSON
- **Needed**: Server returns GeoJSON for agents, markdown for humans

#### 3. **Weather/Environment Services**

- **Problem**: Agentic clients want structured data; human clients want
  narrative
- **Current**: Duplicate endpoints or manual format selection
- **Needed**: Same endpoint, negotiated format

#### 4. **Multi-Agent Orchestration**

- **Problem**: Orchestrator agent composes work for specialist agents; each has
  different capabilities
  - Agent A: has `sampling`, no `elicitation`
  - Agent B: has `elicitation`, no `sampling`
  - Agent C: has both
- **Current**: Server must hardcode behavior for each agent type
- **Needed**: Server reads capabilities upfront, adapts per-agent

#### 5. **Human-in-the-Loop vs Autonomous**

- **Problem**: Same tool used in two modes:
  - Autonomous agent (no human interaction) → needs reasoning hints, structured
    data
  - Human-reviewed loop → needs explanations, confirmation dialogs
- **Current**: Server can't distinguish
- **Needed**: Negotiated capabilities signal mode at initialization

#### 6. **Enterprise API Integration**

- **Problem**: Internal enterprise APIs integrated via MCP need different
  response granularity:
  - AI agents doing batch processing → concise JSON
  - Human analysts reviewing results → detailed reports
- **Current**: API gateway must duplicate data
- **Needed**: Single integration point, content shape negotiated per client

### Why This Matters: Session-Scoped Negotiation

The key insight is that **capability negotiation should happen once at session
initialization**, not per-request:

1. **Performance**: Server optimizes at initialization time, not on every
   request
2. **Correctness**: Client and server agree upfront what formats will be used
3. **Composability**: Tools and resources follow consistent negotiated format
4. **Clarity**: Both parties know what to expect

This matches MCP's design: capabilities are declared at `initialize`, not
per-message.

## Specification

This extension adds two new optional capabilities to the MCP protocol:
`contentNegotiation` for clients and `contentNegotiation` for servers.

### Extension Identifier

```
io.modelcontextprotocol/content-negotiation
```

### Client Extension Settings

Clients implementing this extension MAY declare support via the `extensions`
field in `ClientCapabilities`, with settings containing version and feature
tags:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "sampling": {},
      "extensions": {
        "io.modelcontextprotocol/content-negotiation": {
          "version": "1.0",
          "features": [
            "agent",
            "mcp-capable",
            "!interactive",
            "verbosity=compact",
            "format=json"
          ]
        }
      }
    },
    "clientInfo": {
      "name": "AI Agent Orchestrator",
      "version": "1.0.0"
    }
  }
}
```

### Extension Settings Schema

The extension defines a settings object with the following structure:

```typescript
export interface ContentNegotiationSettings {
  /**
   * Version of the content negotiation protocol.
   * Allows future evolution with version-aware servers.
   * Current version: "1.0"
   */
  version: string;

  /**
   * Feature tags declaring client capabilities and preferences (client only).
   * Servers do not include this field.
   *
   * Tags can be:
   * - Presence: "agent", "sampling" (tag is present)
   * - Negation: "!interactive" (tag is absent)
   * - Equality: "format=json", "verbosity=compact" (tag has value)
   */
  features?: string[]; // Only in client request
}
```

### Server Extension Settings

Servers implementing this extension MAY declare support via the `extensions`
field in `ServerCapabilities`. An empty settings object indicates support:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "logging": {},
      "prompts": { "listChanged": true },
      "resources": { "subscribe": true, "listChanged": true },
      "tools": { "listChanged": true },
      "extensions": {
        "io.modelcontextprotocol/content-negotiation": {}
      }
    },
    "serverInfo": {
      "name": "Weather Service",
      "version": "2.0.0"
    }
  }
}
```

### Feature Tag Registry (Initial v1.0)

| Tag                            | Format   | Meaning                                  | Server Impact                                                                |
| ------------------------------ | -------- | ---------------------------------------- | ---------------------------------------------------------------------------- |
| `agent`                        | Presence | Client is an AI agent (not human)        | Omit prose explanations; enable structured responses                         |
| `human`                        | Presence | Client is a human with UI rendering      | Use markdown; include narrative and examples                                 |
| `mcp-capable`                  | Presence | Client understands MCP protocols/schemas | Server may reference tool/resource names directly; assume technical audience |
| `interactive` / `!interactive` | Negation | Can/cannot present UI dialogs to user    | Gate elicitation-style guidance; no "choose from list" prompts               |
| `sampling`                     | Presence | Client declared sampling capability      | Server may return reasoning hints; include step-by-step logic                |
| `elicitation`                  | Presence | Client declared elicitation capability   | Server may request user input; safe to ask follow-up questions               |
| `roots`                        | Presence | Client declared roots capability         | Server may reference filesystem boundaries                                   |
| `tasks`                        | Presence | Client declared tasks capability         | Server may create long-running operations                                    |
| `verbosity=compact`            | Equality | Prefer concise output                    | Minimize explanatory text; omit examples                                     |
| `verbosity=standard`           | Equality | Default verbosity (balance)              | Normal prose with context                                                    |
| `verbosity=verbose`            | Equality | Prefer detailed output                   | Include rationale, examples, edge cases                                      |
| `format=json`                  | Equality | Prefer JSON/structured output            | Use `structuredContent` in tool responses                                    |
| `format=text`                  | Equality | Prefer plain text                        | Simple prose, no markup                                                      |
| `format=markdown`              | Equality | Prefer markdown                          | Rich formatting with headers, lists, links                                   |
| `x-*`                          | Custom   | Vendor-specific extensions               | Implementation-defined (e.g., `x-openai-format`)                             |

### Feature Tag Predicate Syntax

Feature tags support three syntactic forms (inspired by RFC 2295):

#### 1. **Presence** (simple boolean)

```
"agent"              ← Tag present
"!interactive"       ← Tag absent (negation via !)
```

Server logic: **Include this tag → MUST honor the associated behavior**

#### 2. **Equality** (key=value comparison)

```
"format=json"        ← format equals json
"verbosity=compact"  ← verbosity equals compact
```

Server logic: **If tag is present with this value, apply corresponding
behavior**

#### 3. **Negation of Equality** (key!=value)

```
"format!=xml"        ← format is not xml
```

Server logic: **If tag is present and does NOT equal value, apply behavior**

### How Servers Respond Differently

#### Tool Results (`tools/call`)

**Agent with `format=json`**:

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "content": [],
    "structuredContent": {
      "type": "object",
      "properties": {
        "location": "Bern",
        "temperature_c": 8,
        "humidity_percent": 72,
        "precipitation_probability": 0.3,
        "wind_speed_kmh": 15,
        "uv_index": 2,
        "forecast_summary": "Warming trend expected Thursday"
      }
    }
  }
}
```

**Agent with `sampling` capability**:

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "content": [],
    "structuredContent": {
      "type": "object",
      "properties": {
        "location": "Bern",
        "temperature_c": 8,
        "humidity_percent": 72,
        "precipitation_probability": 0.3,
        "hints": [
          "Temperature is below freezing - user may need winter clothing",
          "30% precipitation chance suggests umbrella is useful",
          "Low UV index (2) means sunscreen not critical"
        ]
      }
    }
  }
}
```

**Human with `format=markdown`**:

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "## Current Weather in Bern\n\n**Temperature**: 8°C (feels like 5°C with wind chill)\n**Humidity**: 72% (comfortable)\n**Conditions**: Mostly cloudy, light precipitation possible (30% chance in next 2 hours)\n**Wind**: 15 km/h from NW\n**UV Index**: 2 (low)\n\n### Forecast\n\nWeather improving this week! Gradually warming trend:\n- **Today**: 8°C, clouds clearing by afternoon\n- **Tomorrow**: 9°C, mostly sunny\n- **Thursday**: 12°C, sunny and pleasant\n\nThis is typical February weather for Bern. Dress in layers!"
      }
    ],
    "structuredContent": null
  }
}
```

**Agent with `!interactive` (no user dialogs)**:

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "content": [],
    "structuredContent": {
      "type": "object",
      "properties": {
        "location": "Bern",
        "temperature_c": 8,
        "status": "success"
      }
    }
  }
}
```

#### Resource Content (`resources/read`)

**Agent with `mcp-capable`**:

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "contents": [
      {
        "uri": "map://features/alpine-valley-1",
        "mimeType": "application/json",
        "text": "{\"type\": \"Feature\", \"geometry\": {\"type\": \"Point\", \"coordinates\": [7.6586, 45.9763]}, \"properties\": {\"name\": \"Alpine Valley\", \"elevation_m\": 3200, \"feature_types\": [\"valley\", \"hiking\", \"scenic\"], \"boundaries\": {\"north\": 45.98, \"south\": 45.96, \"east\": 7.67, \"west\": 7.65}, \"accessibility\": \"moderate\", \"area_km2\": 4.2}}"
      }
    ]
  }
}
```

**Human with `format=markdown`**:

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "contents": [
      {
        "uri": "map://features/alpine-valley-1",
        "mimeType": "text/markdown",
        "text": "# Alpine Valley - Scenic Hiking Destination\n\n## Location\n**Coordinates**: 45.9763°N, 7.6586°E\n**Elevation**: 3,200 meters (10,500 feet)\n**Coverage Area**: Approximately 4.2 km²\n\n## Overview\nA pristine alpine valley surrounded by dramatic peaks and diverse ecosystems. Perfect for hiking, photography, and experiencing alpine flora and fauna.\n\n## Access & Conditions\n- **Difficulty**: Moderate (suitable for intermediate hikers)\n- **Best Season**: June through September (snow-free)\n- **Estimated Hike Time**: 4-6 hours depending on route\n- **Water Sources**: Mountain streams and natural springs\n\n## Features\n- Mountain meadows with wildflowers (peak bloom: July-August)\n- Several hiking trails with varying difficulty levels\n- Natural spring-fed pools\n- 360° views of surrounding peaks\n\n## Planning Tip\nBring layers and sun protection. Weather can change rapidly in alpine areas. Sunrise and sunset offer spectacular photography opportunities."
      }
    ]
  }
}
```

#### Prompt Templates (`prompts/get`)

**Agent with `mcp-capable` and `sampling`**:

```json
{
  "jsonrpc": "2.0",
  "id": 9,
  "result": {
    "messages": [
      {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "For the following tool call, analyze step-by-step:\n1. Parse input parameters\n2. Plan the search strategy\n3. Reason about edge cases\n4. Execute the query\n\nThen call the weather tool."
          }
        ]
      }
    ]
  }
}
```

**Human with `interactive` and `!sampling`**:

```json
{
  "jsonrpc": "2.0",
  "id": 9,
  "result": {
    "messages": [
      {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "## 🌤️ Check the Weather\n\nLet's look up the current weather for a location.\n\n**How to use**:\n1. Tell me a city name (e.g., \"Bern\", \"Zurich\", \"Geneva\")\n2. I'll fetch the latest conditions\n3. We can discuss what to wear or plan activities\n\n### Tips\n- Specify a city in Switzerland for best results\n- Include any specific interests (hiking, skiing, outdoor events)\n- Ask follow-up questions about seasonal conditions\n\nWhat location interests you?"
          }
        ]
      }
    ]
  }
}
```

### Extension Adoption and Graceful Degradation

**Opt-In Nature**:

- This extension is entirely optional. Clients are not required to declare it;
  servers are not required to implement it.
- Absence of this extension does not impact protocol conformance for either
  client or server.

**Graceful Degradation**:

- If a **client declares** `contentNegotiation` but a **server does not
  support** this extension, the server MUST respond with default content
  (identical to non-negotiating behavior).
- If a **server supports** this extension but a **client does not declare** it,
  the server MUST return default content (identical to pre-negotiation
  behavior).
- Extension support is determined by checking the `contentNegotiation`
  capability in the `initialize` handshake.

### Behavioral Requirements

**Requirement 1: MUST Support Backward Compatibility** Servers without the
`contentNegotiation` capability (or clients without declaring it) behave
identically to today's protocol. This extension adds no mandatory requirements
to existing implementations.

**Requirement 2: SHOULD Mirror Declared Capabilities in Features** If a client
declares `sampling: {}` in capabilities, it SHOULD also include `"sampling"` in
the extension's `features` array. This clarifies intent: one field signals
protocol availability, the other signals how to use it. This is a SHOULD-level
recommendation (not enforced).

**Requirement 3: MUST NOT Use Features as Authentication** Feature tags are
**for content shaping only**. They MUST NOT be used to:

- Control access to tools/resources (use authorization)
- Authenticate clients (use credentials)
- Make trust decisions (use signatures/verification)

**Requirement 4: MUST Validate Feature Syntax** Servers SHOULD validate that
feature tags are well-formed (presence, equality, or negation). Invalid syntax
SHOULD be logged but not cause error; unknown tags SHOULD be ignored.

**Requirement 5: SHOULD Prefer Client-Declared Preferences** When a client
declares both `format=json` and uses the `sampling` capability, servers SHOULD
prioritize the explicit format request over capability-inferred format.

**Requirement 6: Server Capability Declaration is Optional** Servers are not
required to declare `contentNegotiation: { honored: true }` in their
capabilities. Clients MUST NOT assume content negotiation is supported unless
explicitly declared.

**Requirement 7: Graceful Fallback** If a client declares unsupported features,
servers MUST respond with default content (same as non-negotiating servers) and
MAY log a warning.

### Examples: Three Initialize Scenarios

#### Scenario 1: AI Agent with Full Capabilities

**Client Request**:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "sampling": {},
      "elicitation": { "form": {}, "url": {} },
      "roots": { "listChanged": true },
      "tasks": {},
      "extensions": {
        "io.modelcontextprotocol/content-negotiation": {
          "version": "1.0",
          "features": [
            "agent",
            "mcp-capable",
            "sampling",
            "elicitation",
            "roots",
            "tasks",
            "verbosity=compact",
            "format=json"
          ]
        }
      }
    },
    "clientInfo": {
      "name": "Claude Agent",
      "version": "1.0.0"
    }
  }
}
```

**Server Response** (supporting this extension):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "logging": {},
      "prompts": { "listChanged": true },
      "resources": { "subscribe": true, "listChanged": true },
      "tools": { "listChanged": true },
      "tasks": { "list": {}, "cancel": {} },
      "extensions": {
        "io.modelcontextprotocol/content-negotiation": {}
      }
    },
    "serverInfo": {
      "name": "Weather Service",
      "version": "2.0.0"
    },
    "instructions": "Content negotiation enabled. Tool results will be JSON-formatted; prompts include reasoning hints."
  }
}
```

#### Scenario 2: Human User in Chat Interface

**Client Request**:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "extensions": {
        "io.modelcontextprotocol/content-negotiation": {
          "version": "1.0",
          "features": [
            "human",
            "!mcp-capable",
            "interactive",
            "verbosity=standard",
            "format=markdown"
          ]
        }
      }
    },
    "clientInfo": {
      "name": "ChatGPT Web Client",
      "version": "1.0.0"
    }
  }
}
```

**Server Response**:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "logging": {},
      "prompts": { "listChanged": true },
      "resources": { "subscribe": true, "listChanged": true },
      "tools": { "listChanged": true },
      "extensions": {
        "io.modelcontextprotocol/content-negotiation": {}
      }
    },
    "serverInfo": {
      "name": "Weather Service",
      "version": "2.0.0"
    },
    "instructions": "Content negotiation enabled. Tool results will be markdown-formatted with narrative explanations."
  }
}
```

#### Scenario 3: Legacy Client (No Content Negotiation)

**Client Request** (old client, no content negotiation extension):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "sampling": {}
    },
    "clientInfo": {
      "name": "OldAgent v0.9",
      "version": "0.9.0"
    }
  }
}
```

**Server Response** (server with extension support):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "logging": {},
      "prompts": { "listChanged": true },
      "resources": { "subscribe": true, "listChanged": true },
      "tools": { "listChanged": true },
      "extensions": {
        "io.modelcontextprotocol/content-negotiation": {}
      }
    },
    "serverInfo": {
      "name": "Weather Service",
      "version": "2.0.0"
    },
    "instructions": "Client does not support content negotiation extension; using default format."
  }
}
```

### Out of Scope for v1.0

The following features are NOT included in v1.0 but may be considered for v1.1+:

1. **Per-request content negotiation**: Feature negotiation happens once at
   initialization, not per-tool-call
2. **Quality degradation factors**: No syntax like `colordepth=3;+0.7` (RFC
   2295's quality boosts)
3. **Variant lists**: Servers don't advertise all possible variants; they select
   based on negotiated features
4. **Remote variant selection algorithm**: RFC 2295's two-phase variant metadata
   exchange
5. **Range predicates**: No syntax like `screenwidth=[1024-1920]`
6. **Feature tag standardization**: v1.0 intentionally leaves most tags to
   community/vendor definition
7. **HTTP Accept-\* header mapping**: This is MCP-specific, not tied to HTTP
   semantics

## Rationale

### Why This Design?

#### 1. **Session-Scoped Negotiation Matches MCP's Architecture**

MCP's capability system is session-scoped: declared once at `initialize`, then
unchanged for the session. Content negotiation fits naturally into this model.

**Alternative rejected**: Per-request negotiation (like HTTP Accept headers)

- Pro: Maximum flexibility per-message
- Con: Adds latency (negotiation overhead on every request)
- Con: Mismatch with MCP's session-scoped capability model
- Con: Complexity for servers (must decide per-request)

#### 2. **Feature Tags Over Opaque Labels**

Rather than generic labels like `agent=true`, clients declare **which MCP
capabilities they actually have**. This is more useful because:

- A server can respond differently to a client with `sampling` than one without
  it
- A server can respond differently to a client with `elicitation` than one
  without it
- The negotiation vocabulary is self-documenting: feature tags directly
  correspond to MCP capabilities

**Alternative rejected**: Generic audience labels (`audience=agent` vs
`audience=human`)

- Pro: Simpler conceptual model
- Con: Doesn't capture nuance (does the agent have sampling? elicitation? both?)
- Con: Servers can't make content decisions based on actual capabilities
- Con: Doesn't scale to new MCP capabilities without spec updates

#### 3. **Flexibility via Equality and Negation Predicates**

Rather than a fixed registry of tags, the syntax supports:

- **Presence**: `agent`, `sampling` (tag exists)
- **Negation**: `!interactive`, `!mcp-capable` (tag is absent)
- **Equality**: `format=json`, `verbosity=compact` (tag has specific value)

This allows vendors and communities to extend the vocabulary without protocol
changes.

**Alternative rejected**: Pre-defined enum of tags

- Pro: Strict spec, no vendor fragmentation
- Con: Inflexible; requires spec updates for new needs
- Con: Doesn't support vendor-specific extensions

#### 4. **Inspired by RFC 2295 but Simpler**

RFC 2295 defined a full two-phase content negotiation system with variant
metadata, remote algorithms, entity tags, and caching semantics. We adopted its
**core insight** (clients declare capabilities → servers select variants) but
simplified to fit MCP:

- **Adopted from RFC 2295**:
  - Feature tag concept (presence, negation, equality predicates)
  - Session-scoped negotiation metadata
  - Graceful fallback for unsupported features

- **Not adopted from RFC 2295**:
  - Two-phase variant list exchange (unnecessary in stateful MCP)
  - Remote variant selection algorithm (complexity not justified)
  - Quality degradation factors and range predicates (v1.0 scope)
  - Caching/entity tag extensions (not applicable to MCP's primitives)

**Why simplification works**: MCP is stateful and session-based (unlike HTTP),
eliminating the need for RFC 2295's complex caching and two-phase negotiation.

### Handling Existing Use Cases

#### Journey Service

- **Before**: One tool returns prose, needs to be parsed by agent
- **After**: Agent declares `format=json`, tool returns structured data directly

#### Geospatial/Mapping Service

- **Before**: Server returns descriptions, agent wastes context parsing text
- **After**: Agent declares `format=json, mcp-capable`, server returns GeoJSON
  with coordinates and properties

#### Multi-Agent Orchestration

- **Before**: Orchestrator doesn't know what format each specialist agent needs
- **After**: Each agent declares its features; server adapts per-connection

#### Autonomous vs Human-in-the-Loop

- **Before**: Server hardcodes behavior or has two separate tool sets
- **After**: Mode specified via negotiated features; single tool serves both use
  cases

### Why Not Annotations?

MCP already has `annotations` on message blocks. Why not extend that?

**Answer**: Annotations solve a different problem:

- Annotations are **per-message-block**, applied by servers dynamically
- Content negotiation is **session-scoped**, declared by clients upfront

**Trade-off analysis**:

- Annotations: More flexible (can vary per-response), but server must decide
  on-the-fly
- Content negotiation: Less flexible (static per-session), but enables server
  optimization at init time

These mechanisms are complementary, not competing. A server could:

1. Negotiate content format at initialization (via `contentNegotiation`)
2. Add per-block annotations for fine-grained guidance (via existing
   `annotations`)

### Why Not Separate Experimental Features?

The `experimental` capability exists for non-standard features. Why standardize
this?

**Answer**: Content adaptation is broadly useful (not niche), and today's
workarounds are ad-hoc:

- No one else's server understands your custom `experimental` features
- Every server invents its own dialect
- Clients have no standard way to declare preferences

This SEP provides a standard vocabulary for content negotiation that works
across all servers.

### RFC 2295 Mapping Table

How does this SEP relate to RFC 2295's concepts?

| RFC 2295 Concept                                                | MCP Implementation                             | Adopted/Adapted/Rejected                            |
| --------------------------------------------------------------- | ---------------------------------------------- | --------------------------------------------------- |
| Variant metadata (list of alternatives)                         | Server selects directly, no metadata exchange  | **Rejected** (stateful session eliminates need)     |
| Feature tags (presence, negation, equality)                     | Same syntax in `features[]`                    | **Adopted**                                         |
| Accept-Features header (client declares capabilities)           | `contentNegotiation.features[]` in initialize  | **Adapted** (session-scoped instead of per-request) |
| Transparent negotiation (client describes, server selects)      | Core of this proposal                          | **Adopted**                                         |
| Two-phase exchange (client asks for variant list, then selects) | Single-phase: server decides at initialization | **Rejected** (MCP's stateful model doesn't need it) |
| Quality degradation factors (e.g., `colordepth=3;+0.7`)         | Not in v1.0                                    | **Deferred**                                        |
| Range predicates (e.g., `screenwidth=[1024-1920]`)              | Not in v1.0                                    | **Deferred**                                        |
| Remote algorithm versions (negotiated at session start)         | `contentNegotiation.version` in capability     | **Adapted**                                         |
| Entity tag validation (`ETag: "abc;vlv1"`)                      | Not applicable to MCP                          | **Not applicable**                                  |
| Caching strategy (variant list freshness)                       | Not applicable to MCP                          | **Not applicable**                                  |
| TCN response header                                             | No per-message headers in MCP                  | **Not applicable**                                  |

### Prior Art: SEP-2053 Server Variants

[SEP-2053](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2053)
(Sambhav Kothari, January 2026) addresses the same root problem — MCP servers
cannot adapt to different client types — but from the opposite direction.
Rather than clients declaring capabilities, servers advertise named
configurations (`claude-optimized`, `compact`, `automation-agent`) and clients
select among them per-request via `_meta`.

**Key differences**:

| Dimension | This extension | SEP-2053 |
| --- | --- | --- |
| Who drives selection | Client declares capabilities; server adapts | Server ranks variants; client picks |
| What varies | Response content (format, verbosity, structure) | Tool/resource surface (which tools exist, their schemas) |
| Negotiation scope | Session-scoped once at `initialize` | Per-request via `_meta` on every call |
| Semantic grounding | Tags tied to MCP protocol capabilities | Open-ended key-value hint strings |
| Per-request overhead | None | `_meta` on every request; variant-scoped cursors and subscriptions |

**What SEP-2053 addresses that this extension does not**: Exposing entirely
different tool catalogs to different agents (e.g., a `code-review` variant with
PR tools vs. an `automation` variant with pipeline tools). This extension does
not change which tools exist — only how their results are formatted.

**What this extension addresses that SEP-2053 does not**: Precise,
protocol-grounded semantics. Feature tags like `sampling`, `elicitation`, and
`roots` correspond directly to MCP capabilities the client has declared. SEP-2053's
hints (`modelFamily`, `useCase`, vendor keys) are open-ended strings requiring
heuristic or LLM-based matching. Notably, both proposals independently arrived
at the same vocabulary axes (`compact`/`standard`/`verbose` verbosity,
`agent`/`interactive` audience) — confirming these are the right dimensions for
client-server adaptation.

**Security model**: This extension enforces that feature tags control *what is
sent*, never *whether to send it* (no access control via tags). SEP-2053 uses
variants for access gating (`"security-readonly"` variant, `accessLevel` hints),
which can create authorization gaps if variant identity is not treated as a
security boundary.

**Complementarity**: The two proposals operate at different layers and are not
mutually exclusive. A server could implement both — SEP-2053 variants to select
*which tool surface* to expose, and this extension to shape *how results are
formatted* within that surface.

## Backward Compatibility

This extension is **fully backward compatible** with the core MCP protocol,
following SEP-2133 principles:

### Core Protocol Compatibility

1. **Extension is optional**: Neither clients nor servers are required to
   declare or support this extension
2. **Non-breaking**: This extension adds only new optional capabilities; it
   removes, renames, or changes no existing protocol elements
3. **No mandatory fields**: All fields introduced by this extension are optional
4. **Graceful degradation**: Clients and servers without this extension continue
   to function without modification

### Migration Path for Ecosystem

**Phase 1 (Immediate)**: This extension is submitted as Draft

- Early adopters can implement content negotiation in their servers and clients
- Ecosystem gains experience with feature declarations
- Feature tag semantics emerge through community use

**Phase 2 (6-12 months)**: Extension reaches Final status (if accepted)

- Wider adoption; major servers implement negotiation
- SDKs provide helper functions for feature declaration
- Best practices and conventions stabilize

**Phase 3 (2+ years)**: Potential future standardization (optional)

- Successful extensions may transition to core protocol features
- This extension may remain optional indefinitely (as many extensions do)
- Not all extensions require or should have core protocol status

### Existing Implementations

Existing servers and clients require **zero changes** to remain compatible:

- They may safely ignore the `contentNegotiation` capability if declared
- They may omit it from their response
- They will return identical content to all clients

### Versioning and Evolution

This extension follows independent versioning (separate from core protocol
versions). Future versions may:

- Add new feature tags
- Refine negotiation semantics
- Introduce new use cases

Breaking changes MUST use a new extension identifier (e.g.,
`io.modelcontextprotocol/content-negotiation-v2`).

## Security Implications

### Risk 1: Feature Tag Spoofing

**Threat**: A malicious client declares false features to trick a server into
revealing privileged data.

**Example**: Client declares `!interactive` (claiming no user dialog capability)
to bypass elicitation-based confirmations.

**Mitigation**:

- Feature tags are for **content shape only**, not access control
- Authorization remains the sole mechanism for access decisions
- Servers MUST NOT use feature tags to bypass user consent
- Principle: Feature tags optimize format, not permissions

**Guideline**: Treat feature tags as hints from an untrusted client. Make
authorization decisions based on credentials, not features.

### Risk 2: Tag Injection

**Threat**: A client sends invalid feature tag syntax that crashes parser.

**Example**: Client sends `features: ["format=\n"]` to inject newlines.

**Mitigation**:

- Servers SHOULD validate feature tag syntax (presence, equality, negation
  forms)
- Invalid tags SHOULD be ignored, not cause error
- No security boundary depends on parsing correctness
- Principle: Graceful degradation

**Guideline**: Validate rigorously, fail gracefully.

### Risk 3: Semantic Ambiguity

**Threat**: Different servers interpret `format=json` differently, leading to
unexpected content.

**Example**: Server A returns bare JSON object. Server B wraps it in
`{ data: {...} }`. Client expects one, gets the other.

**Mitigation**:

- Feature tag semantics SHOULD be documented per-server
- In v1.0, tags are community-defined with no enforcement
- Expect fragmentation until a feature tag registry matures
- Clients SHOULD test actual server behavior, not assume standard meaning

**Guideline**: Community communication and documentation reduce semantic
confusion.

### Risk 4: No Authentication via Features

**Threat**: Developer confuses feature tags with identity/authentication.

**Explicit Statement**: Feature tags MUST NOT be used for:

- Authentication (use credentials, API keys, certificates)
- Authorization (use access control lists)
- Trust decisions (use signatures, verification)
- Rate limiting (use auth tokens)

**Guideline**: Separate content negotiation (what format?) from authentication
(who are you?). Never conflate the two.

### Risk 5: Privacy of Declared Features

**Consideration**: When a client declares features, it reveals:

- Protocol capabilities (sampling, elicitation, roots, tasks)
- Content preferences (format, verbosity)
- Client type (agent vs human)

**Mitigation**:

- Feature declarations occur at initialization (not per-request)
- Features are visible to the connected server only
- Features do NOT leak across servers (host mediates isolation)
- Transport security (TLS, authentication) protects initialization exchange

**Guideline**: Treat feature declarations as session metadata. Protect
initialization exchanges with transport security.

## Reference Implementation

This section describes how implementations can adopt content negotiation.

### SDK Implementation Guidance

SDKs implementing this extension SHOULD:

- Disable the extension by default; require explicit opt-in by applications
- Document supported extension features in SDK documentation
- Provide helper functions for parsing and matching feature tags (see example
  below)
- Consider packaging the extension separately or behind a feature flag

SDKs are under no obligation to implement this extension or accept contributed
implementations. Extension support is not required for protocol conformance.

### TypeScript Schema Changes

Per SEP-2133, extensions are advertised via the `extensions` field in
capabilities. This extension defines a settings object for use in that field:

```typescript
/**
 * Settings for the content-negotiation extension.
 * Declared by clients in extensions["io.modelcontextprotocol/content-negotiation"]
 */
export interface ContentNegotiationSettings {
  /**
   * Version of the content negotiation protocol.
   * Enables future evolution while maintaining compatibility.
   * Current version: "1.0"
   */
  version: string;

  /**
   * Feature tags declaring client capabilities and preferences.
   * Only present in client request, not in server response.
   *
   * Tags can be:
   * - Presence: "agent", "sampling", "interactive" (tag is present)
   * - Negation: "!interactive", "!mcp-capable" (tag is absent)
   * - Equality: "format=json", "verbosity=compact" (tag has specific value)
   *
   * Standard tags:
   * - "agent" | "human": Client type
   * - "mcp-capable": Client understands MCP protocols
   * - "interactive" | "!interactive": Can present UI to user
   * - "sampling", "elicitation", "roots", "tasks": Declared MCP capabilities
   * - "verbosity=compact|standard|verbose": Preferred response length
   * - "format=json|text|markdown": Preferred output format
   * - "x-*": Vendor-specific extensions
   *
   * Servers SHOULD ignore unknown tags and log warnings for unsupported features.
   */
  features?: string[];
}

// No changes to existing ClientCapabilities or ServerCapabilities
// Extension support is advertised via the existing "extensions" field:
//
// Client: capabilities.extensions["io.modelcontextprotocol/content-negotiation"] = { version, features }
// Server: capabilities.extensions["io.modelcontextprotocol/content-negotiation"] = {} (empty object)
```

### Helper: Parse and Match Features

Servers can implement a helper to parse and match feature tags:

```typescript
type FeatureMatch =
  | { type: 'presence'; tag: string }
  | { type: 'negation'; tag: string }
  | { type: 'equality'; key: string; value: string }
  | { type: 'error'; reason: string };

function parseFeature(tag: string): FeatureMatch {
  if (tag.startsWith('!')) {
    return { type: 'negation', tag: tag.substring(1) };
  }

  if (tag.includes('=')) {
    const [key, value] = tag.split('=', 2);
    if (!key || !value) return { type: 'error', reason: 'Invalid equality' };
    return { type: 'equality', key, value };
  }

  if (/^[a-zA-Z0-9_-]+$/.test(tag)) {
    return { type: 'presence', tag };
  }

  return { type: 'error', reason: 'Invalid feature syntax' };
}

function hasFeature(features: string[], feature: string): boolean {
  return features.includes(feature);
}

function hasEqualityFeature(
  features: string[],
  key: string,
  value: string,
): boolean {
  return features.includes(`${key}=${value}`);
}

function lackFeature(features: string[], feature: string): boolean {
  return !features.includes(feature);
}

// Usage:
const serverCapabilities =
  clientCapabilities.extensions?.[
    'io.modelcontextprotocol/content-negotiation'
  ];
const clientFeatures = serverCapabilities?.features || [];

if (hasFeature(clientFeatures, 'agent')) {
  // Return JSON-formatted response
}

if (hasEqualityFeature(clientFeatures, 'format', 'json')) {
  // Use structuredContent, not prose content
}

if (lackFeature(clientFeatures, 'interactive')) {
  // Omit elicitation-style prompts
}
```

### Tool Response Example: Full Agent vs Human

**Weather tool implementation that respects negotiation**:

```typescript
async function getWeatherToolResponse(
  location: string,
  clientCapabilities: ClientCapabilities,
): Promise<ToolResultContent> {
  const weatherData = await fetchWeather(location);

  // Extract features from extension settings
  const extensionSettings =
    clientCapabilities.extensions?.[
      'io.modelcontextprotocol/content-negotiation'
    ];
  const clientFeatures = extensionSettings?.features || [];

  const isAgent = hasFeature(clientFeatures, 'agent');
  const wantsJson = hasEqualityFeature(clientFeatures, 'format', 'json');
  const hasSampling = hasFeature(clientFeatures, 'sampling');
  const verbosity = getEqualityValue(clientFeatures, 'verbosity') || 'standard';

  if (isAgent && wantsJson) {
    // Return structured JSON for agent processing
    return {
      content: [],
      structuredContent: {
        type: 'object',
        properties: {
          location: weatherData.location,
          temperature_c: weatherData.temperature,
          humidity_percent: weatherData.humidity,
          precipitation_probability: weatherData.precipitation,
          wind_speed_kmh: weatherData.windSpeed,
          uv_index: weatherData.uvIndex,
          ...(hasSampling && {
            reasoning_hints: [
              `Temperature ${weatherData.temperature}°C suggests appropriate clothing`,
              `${weatherData.precipitation}% precipitation: ${weatherData.precipitation > 50 ? 'bring umbrella' : 'umbrella optional'}`,
            ],
          }),
        },
      },
    };
  } else if (!isAgent) {
    // Return rich markdown for human reading
    const markdown = formatWeatherMarkdown(weatherData, verbosity);
    return {
      content: [{ type: 'text', text: markdown }],
      structuredContent: null,
    };
  }

  // Fallback: both content and structured for safety
  return {
    content: [
      { type: 'text', text: formatWeatherMarkdown(weatherData, 'standard') },
    ],
    structuredContent: toStructuredContent(weatherData),
  };
}
```

### Expected Behavior: Negotiating Server

When a client initializes with content negotiation features:

1. **Parse Features**: Extract and validate feature tags from
   `contentNegotiation.features[]`
2. **Store in Session Context**: Keep parsed features available for all
   subsequent requests
3. **Vary Tool Responses**: Apply feature rules to tool results
4. **Vary Resource Content**: Apply feature rules to resource reads
5. **Vary Prompt Templates**: Apply feature rules to prompt generation
6. **Log Negotiations**: Debug log showing which features client requested

### Testing Content Negotiation

**Test Case 1: Agent requesting JSON**:

```
Client features: ["agent", "format=json"]
Tool: get_weather
Expected: structuredContent with JSON object, empty content[]
```

**Test Case 2: Human requesting markdown**:

```
Client features: ["human", "format=markdown"]
Tool: get_weather
Expected: content[] with markdown text, structuredContent empty
```

**Test Case 3: Legacy client (no negotiation)**:

```
Client features: [] (not declared)
Tool: get_weather
Expected: Server default format (backwards compatible)
```

**Test Case 4: Invalid feature syntax**:

```
Client features: ["@#$%", "format==json"]
Server behavior: Log warning, ignore invalid tags, use defaults
```

## Additional Considerations

### Performance Implications

**Positive**:

- Servers can optimize at initialization time, not per-request
- Tool results are smaller (JSON vs prose)
- Agent reasoning more efficient (structured data, no parsing)

**Negligible**:

- Feature tag parsing is O(n) where n=number of features (~5-20 typical)
- One-time cost at initialization

### Testing Plan

1. **Unit Tests**: Feature tag parsing logic
2. **Integration Tests**: Server with multiple clients (agent and human
   simultaneously)
3. **Interop Tests**: Client declares features, server honors them
4. **Backward Compat Tests**: Clients without negotiation still work
5. **Variant Testing**: Same request with different feature combinations

### Alternatives Considered and Rejected

#### Alternative 1: Per-Message Content Negotiation (via Accept-like Headers)

**Proposal**: Extend messages with `Accept-Format: json` on every tool call.

**Rejected because**:

- Adds latency (negotiation overhead per-request)
- Incompatible with MCP's stateless capability model
- HTTP analogy doesn't map cleanly (MCP is JSON-RPC, not HTTP)
- Complexity for servers (must decide per-message)

#### Alternative 2: Separate Tool Endpoints for Each Format

**Proposal**: Create `get_weather_json`, `get_weather_markdown` instead of one
tool.

**Rejected because**:

- Duplication and maintenance burden
- Confuses LLM (many similar tools)
- Doesn't scale (N formats × M tools = N×M endpoints)

#### Alternative 3: Server Publishes Variant Metadata (Like RFC 2295)

**Proposal**: At initialization, server sends list of all supported content
variants.

**Rejected because**:

- Unnecessary complexity (MCP's stateful model doesn't need two-phase exchange)
- Doesn't match MCP's existing patterns (capabilities are simple declarations)
- Harder to implement and test

#### Alternative 4: Generic Audience Labels Only

**Proposal**: Simple `audience: "agent" | "human"` capability.

**Rejected because**:

- Loses information (doesn't capture which capabilities client has)
- Doesn't enable nuanced responses (sampling-aware vs sampling-agnostic)
- Can't express vendor-specific preferences
- Inflexible (requires spec updates for new capabilities)

### Open Questions for Community

1. **Feature Tag Standardization**: Should this SEP define a canonical registry
   of standard tags (v1.0), or leave it to emerge organically?
   - Current approach: Informal registry in this document
   - Alternative: Separate Informational SEP with official tag definitions

2. **Vendor Prefix Conventions**: Should we recommend specific prefixes for
   vendor tags (e.g., `x-openai-`, `x-anthropic-`)?

3. **Quality Factors (v1.1)**: Should future versions support RFC 2295's quality
   degradation syntax for feature combinations?

## Extension Governance

Following SEP-2133, this extension is proposed for official MCP status:

- **Target Repository**:
  `https://github.com/modelcontextprotocol/ext-content-negotiation`
- **Extension Maintainers**: (To be appointed by core MCP maintainers upon
  acceptance)
- **Working Group**: Interested community members may form a working group to
  guide development and gather feedback
- **Review Process**: Per SEP-2133 Extensions Track; requires at least one
  reference implementation in an official SDK prior to final review

## Acknowledgments

This extension is inspired by:

- **RFC 2295 (Transparent Content Negotiation in HTTP)**: For the core concept
  of clients declaring capabilities and servers selecting variants
- **SEP-2133 (Extensions)**: For the extension framework and governance model
- **MCP Specification**: For the capability negotiation model and session-scoped
  design patterns
- **Real use cases** from journey, mapping, weather, and multi-agent
  orchestration services: For motivating the problem
- **Community discussion** on MCP extensibility: For validating the need for
  content adaptation

---

**Created**: February 22, 2026 **Status**: Draft **Type**: Extensions Track
**Next Steps**:

1. Community feedback on extension approach and feature tag vocabulary
2. Reference implementation in official MCP SDKs
3. Submission to core maintainers for review under Extensions Track process
   (SEP-2133)
