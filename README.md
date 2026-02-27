[![Status: Draft](https://img.shields.io/badge/Status-Draft-yellow?style=flat-square)](#status)
[![Version: v0.9.6](https://img.shields.io/badge/Version-v0.9.6-blue?style=flat-square)](https://github.com/schlpbch/ext-content-negotiation/releases)
[![Type: Extensions Track](https://img.shields.io/badge/Type-Extensions%20Track-purple?style=flat-square)](#status)
[![Extension ID: io.modelcontextprotocol/content-negotiation](https://img.shields.io/badge/Extension-io.modelcontextprotocol/content--negotiation-brightblue?style=flat-square)](#about-this-extension)

## Overview

This proposal introduces a **content negotiation mechanism** for MCP following
[SEP-2133: Extensions](https://modelcontextprotocol.io/community/seps/2133-extensions),
allowing servers to adapt response formats based on client-declared
capabilities. Inspired by
[RFC 2295](https://www.rfc-editor.org/rfc/rfc2295.html), adapted for MCP's
session-scoped architecture.

![Content Negotiation Diagram](content-negotiation-diagram.png)

Servers can return **JSON for agents**, **markdown for humans**, **reasoning
hints** for agents with sampling, and gate **interactive dialogs** on
elicitation capability.

## Problem Statement

MCP servers cannot distinguish between AI agents (need structured data), humans
(need prose), and agents with different capabilities (sampling, elicitation,
roots, tasks). This forces developers toward:

1. **Duplicate tools** (`weather_json` + `weather_prose`)
2. **Unreliable heuristics** (guessing from `clientInfo.name`)
3. **Bloated responses** (supporting all formats simultaneously)

## Solution

Clients declare the extension during `initialize`:

```json
{
  "capabilities": {
    "extensions": {
      "io.modelcontextprotocol/content-negotiation": {
        "version": "1.0",
        "features": ["agent", "sampling", "format=json", "verbosity=compact"]
      }
    }
  }
}
```

Servers advertise support with an empty object in their capabilities, then
respond with content optimized for the declared features.

## Key Features

- [x] **[RFC 2295](https://www.rfc-editor.org/rfc/rfc2295.html)-Inspired**: HTTP
      transparent content negotiation adapted to MCP
- [x] **Session-Scoped**: Negotiation happens once at initialization, not
      per-request
- [x] **Capability-Driven**: Feature tags mirror client's actual MCP
      capabilities
- [x] **Flexible Predicates**: Supports presence, negation, and equality syntax
- [x] **Fully Backward Compatible**: Zero breaking changes, optional feature
- [x] **Secure**: Feature tags control content shape only, never auth/access

## Real-World Use Cases

| Use Case                  | Problem                                             | Solution                                    |
| ------------------------- | --------------------------------------------------- | ------------------------------------------- |
| Journey Service           | Agent needs structured data; human needs itinerary  | One tool, negotiated format                 |
| Geospatial/Mapping        | Rich descriptions vs. coordinates/boundaries        | GeoJSON for agents, markdown for humans     |
| Multi-Agent Orchestration | Different specialist agents, different capabilities | Server adapts per agent's init capabilities |
| Weather/Environment       | Same data, different consumers                      | Negotiated format eliminates duplication    |

## Main Document

Full specification in
**[ext-content-negotiation.md](ext-content-negotiation.md)** covering:

- **Abstract** - Problem and solution overview
- **Motivation** - 6 real-world use cases + 4 workaround analyses
- **Specification** - New capabilities, feature tags, examples
- **Rationale** - Design decisions, RFC 2295 mapping, alternatives considered
  (including comparison with [SEP-2053: Server Variants](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2053))
- **Backward Compatibility** - No breaking changes, migration path
- **Security Implications** - 5 risks analyzed with mitigations
- **Reference Implementation** - TypeScript schemas, helper functions, examples
  (see also [TYPESCRIPT_REFERENCE.md](TYPESCRIPT_REFERENCE.md),
  [FASTMCP_REFERENCE.md](FASTMCP_REFERENCE.md),
  [SPRING_AI_REFERENCE.md](SPRING_AI_REFERENCE.md),
  [RUST_REFERENCE.md](RUST_REFERENCE.md), and
  [GO_REFERENCE.md](GO_REFERENCE.md) for full worked examples)

## Feature Tags (v1.0 Registry)

| Tag                                            | Meaning                   | Server Impact                           |
| ---------------------------------------------- | ------------------------- | --------------------------------------- |
| `agent` / `human`                              | Client type               | Content format preference               |
| `mcp-capable`                                  | Understands MCP protocols | Can reference tool/resource names       |
| `interactive` / `!interactive`                 | Can present UI            | Gate elicitation-style prompts          |
| `sampling` / `elicitation` / `roots` / `tasks` | Declared MCP capabilities | Include reasoning hints, allow requests |
| `verbosity=compact\|standard\|verbose`         | Response length           | Omit/expand explanations                |
| `format=json\|text\|markdown`                  | Output format             | Use structuredContent vs prose          |
| `x-*`                                          | Vendor-specific           | Custom implementations                  |

## Quick Start

### Server Side

```typescript
const features =
  client.capabilities.extensions?.[
    'io.modelcontextprotocol/content-negotiation'
  ]?.features ?? [];

if (features.includes('agent') && features.includes('format=json')) {
  return { structuredContent: { temperature_c: 8, humidity_percent: 72 } };
} else {
  return { content: [{ type: 'text', text: "It's 8°C with 72% humidity." }] };
}

// Advertise support
const serverCapabilities = {
  extensions: { 'io.modelcontextprotocol/content-negotiation': {} },
};
```

### Client Side

```typescript
const capabilities = {
  sampling: {},
  extensions: {
    'io.modelcontextprotocol/content-negotiation': {
      version: '1.0',
      features: [
        'agent',
        'sampling',
        'mcp-capable',
        'format=json',
        'verbosity=compact',
      ],
    },
  },
};
```

## Security

Feature tags optimize **what** the server sends, not **whether** it should send
it.

| Not for         | Use instead             |
| --------------- | ----------------------- |
| Authentication  | Credentials             |
| Authorization   | Access control lists    |
| Trust decisions | Signatures/verification |
| Rate limiting   | Auth tokens             |

## How to Review

1. **Abstract** (2 min) - Problem and solution
2. **Motivation** (10 min) - Validate use cases
3. **Specification** (15 min) - Ensure implementability
4. **Rationale** (5 min) - Design decisions
5. **Security** (5 min) - Risk mitigations
6. **Reference Implementation** (10 min) - Validate feasibility

## Contributing

1. **Review** [ext-content-negotiation.md](ext-content-negotiation.md)
2. **Feedback** via
   [Issues](https://github.com/schlpbch/agentic-content-negotiation/issues) or
   [MCP Community Discussions](https://github.com/modelcontextprotocol/modelcontextprotocol/discussions)
3. **Test** reference implementations:
   [TypeScript](TYPESCRIPT_REFERENCE.md) · [Python/FastMCP](FASTMCP_REFERENCE.md) · [Java/Spring AI](SPRING_AI_REFERENCE.md) · [Rust/rmcp](RUST_REFERENCE.md) · [Go](GO_REFERENCE.md)
4. **Suggest** improvements to feature tags, semantics, or design

## Links

- **MCP Specification**: https://modelcontextprotocol.io/specification
- **SEP-2133: Extensions**:
  https://modelcontextprotocol.io/community/seps/2133-extensions
- **RFC 2295**: https://www.rfc-editor.org/rfc/rfc2295.html
- **MCP Community**:
  https://github.com/modelcontextprotocol/modelcontextprotocol

## Status

| Field        | Value                                         |
| ------------ | --------------------------------------------- |
| Extension ID | `io.modelcontextprotocol/content-negotiation` |
| Status       | Draft                                         |
| Type         | Extensions Track                              |
| Created      | February 22, 2026                             |
| Phase        | Community feedback and design refinement      |

---

Reference implementations: [TYPESCRIPT_REFERENCE.md](TYPESCRIPT_REFERENCE.md) ·
[FASTMCP_REFERENCE.md](FASTMCP_REFERENCE.md) ·
[SPRING_AI_REFERENCE.md](SPRING_AI_REFERENCE.md) ·
[RUST_REFERENCE.md](RUST_REFERENCE.md) ·
[GO_REFERENCE.md](GO_REFERENCE.md)

Supporting analysis: [RFC2295_ANALYSIS.md](RFC2295_ANALYSIS.md) ·
[MCP_ANALYSIS.md](MCP_ANALYSIS.md) ·
[MCP_PROPOSAL_GUIDE.md](MCP_PROPOSAL_GUIDE.md) ·
[SEP2053_COMPARISON.md](SEP2053_COMPARISON.md)

This proposal follows MCP's licensing model. For questions, open an issue or
discussion.
