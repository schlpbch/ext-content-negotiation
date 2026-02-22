# Agentic Content Negotiation for MCP

A formal Specification Enhancement Proposal (SEP) for transparent content negotiation in the Model Context Protocol (MCP), inspired by RFC 2295.

## Overview

This project proposes a **content negotiation mechanism** that allows MCP servers to adapt response formats based on client-declared capabilities. Instead of serving one-size-fits-all responses, servers can:

- Return **JSON for AI agents** that need structured data
- Return **markdown for humans** who need narrative explanations
- Return **reasoning hints for agents with sampling** vs simple results for agents without
- Gate **interactive dialogs** based on client's elicitation capability

## Problem Statement

MCP servers currently cannot distinguish between:
- AI agents that need structured data for processing
- Humans that need narrative explanations
- Agents with different protocol capabilities (sampling, elicitation, roots, tasks)

This leads to either:
1. **Duplicate tools** (weather_json + weather_prose)
2. **Unreliable heuristics** (guessing based on clientInfo.name)
3. **Bloated responses** (supporting all formats simultaneously)

## Solution: Transparent Content Negotiation

Clients declare capabilities during `initialize`:

```json
{
  "contentNegotiation": {
    "version": "1.0",
    "features": [
      "agent",
      "sampling",
      "format=json",
      "verbosity=compact"
    ]
  }
}
```

Servers respond with content optimized for those capabilities:

```json
{
  "structuredContent": {
    "type": "object",
    "properties": {
      "temperature_c": 8,
      "humidity_percent": 72
    }
  }
}
```

## Key Features

- [x] **RFC 2295-Inspired**: Borrows concepts from HTTP transparent content negotiation, adapted to MCP
- [x] **Session-Scoped**: Negotiation happens once at initialization, not per-request
- [x] **Capability-Driven**: Feature tags mirror client's actual MCP capabilities
- [x] **Flexible Predicates**: Supports presence, negation, and equality syntax
- [x] **Fully Backward Compatible**: Zero breaking changes, optional feature
- [x] **Secure**: Feature tags are for content shape only, never for auth/access control

## Real-World Use Cases

### 1. SBB Journey Service
- **Problem**: Agent needs structured train data; human needs readable itinerary
- **Solution**: One tool, negotiated format (JSON for agent, markdown for human)

### 2. Swiss Tourism
- **Problem**: Sight descriptions are rich narratives; agents need just coordinates/categories
- **Solution**: Server returns `application/json` for agents, `text/markdown` for humans

### 3. Multi-Agent Orchestration
- **Problem**: Different specialist agents have different capabilities
- **Solution**: Orchestrator reads each agent's capabilities at init, server adapts responses

### 4. Weather/Environment Services
- **Problem**: Same data used by agents (need JSON) and humans (need prose)
- **Solution**: Negotiated format eliminates duplication

## Document Structure

### Main Document: `SEP-DRAFT-agent-content-negotiation.md`

**1,210 lines** covering:

- **Abstract** - Problem and solution overview
- **Motivation** - 6 real-world use cases + 4 workaround analyses
- **Specification** - New capabilities, feature tags, examples
- **Rationale** - Design decisions, RFC 2295 mapping, alternatives considered
- **Backward Compatibility** - No breaking changes, migration path
- **Security Implications** - 5 risks analyzed with mitigations
- **Reference Implementation** - TypeScript schemas, helper functions, examples
- **Additional Sections** - Performance, testing, open questions

## Feature Tags (v1.0 Registry)

| Tag | Meaning | Server Impact |
|---|---|---|
| `agent` / `human` | Client type | Content format preference |
| `mcp-capable` | Understands MCP protocols | Can reference tool/resource names |
| `interactive` / `!interactive` | Can present UI | Gate elicitation-style prompts |
| `sampling` / `elicitation` / `roots` / `tasks` | Declared MCP capabilities | Include reasoning hints, allow requests |
| `verbosity=compact\|standard\|verbose` | Response length | Omit/expand explanations |
| `format=json\|text\|markdown` | Output format | Use structuredContent vs prose |
| `x-*` | Vendor-specific | Custom implementations |

## How to Review

### For MCP Maintainers & Contributors

1. **Read the Abstract** (2 min) - Understand the problem and solution
2. **Review Motivation** (10 min) - Validate the real-world use cases
3. **Examine Specification** (15 min) - Ensure implementability
4. **Check Rationale** (5 min) - Verify design decisions
5. **Assess Security** (5 min) - Confirm risks are mitigated
6. **Review Reference Implementation** (10 min) - Validate feasibility

### Key Questions to Answer

- ✓ Does this solve real problems?
- ✓ Is the specification clear and implementable?
- ✓ Are security implications properly addressed?
- ✓ Is backward compatibility truly maintained?
- ✓ Are there better alternatives?

### Expected Timeline

- **Draft Phase**: Community feedback and refinement (this stage)
- **In-Review Phase**: Core Maintainer review + reference implementation
- **Final Phase**: Approval and integration into MCP spec

## Quick Start for Implementation

### Server Side

```typescript
// Parse client features at initialization
const features = client.capabilities.contentNegotiation?.features || [];

// Check client type
const isAgent = features.includes("agent");
const wantsJson = features.includes("format=json");

// Vary tool responses
if (isAgent && wantsJson) {
  return { structuredContent: {...} };  // JSON
} else {
  return { content: [{type: "text", text: "..."}] };  // Markdown
}
```

### Client Side

```typescript
// Declare capabilities at initialization
const capabilities = {
  sampling: {},
  contentNegotiation: {
    version: "1.0",
    features: [
      "agent",
      "sampling",
      "mcp-capable",
      "format=json",
      "verbosity=compact"
    ]
  }
};
```

## Security & Privacy

### What Features Are NOT Used For

- ❌ Authentication (use credentials instead)
- ❌ Authorization (use access control lists)
- ❌ Trust decisions (use signatures/verification)
- ❌ Rate limiting (use auth tokens)

### What Features ARE Used For

- ✅ Content format selection (JSON vs markdown)
- ✅ Response length (verbosity)
- ✅ Response hints (reasoning steps)

**Principle**: Feature tags optimize *what the server sends*, not *whether the server should send it*.

## Examples Included

### Tool Response Variants

- Agent requesting JSON → `{ structuredContent: {...} }`
- Agent with sampling → `{ structuredContent: { hints: [...] } }`
- Agent without interactive → `{ structuredContent: {...} }` (no dialogs)
- Human requesting markdown → `{ content: [{type: "text", text: "..."}] }`

### Initialize Scenarios

- **Scenario 1**: AI agent with full capabilities
- **Scenario 2**: Human user in chat interface
- **Scenario 3**: Legacy client (no negotiation)

### RFC 2295 Mapping

13-row table showing:
- What concepts were **adopted** from RFC 2295
- What were **adapted** for MCP
- What were **rejected** as unnecessary
- What was **deferred** to v1.1+

## Contributing

This is a **draft proposal** seeking community feedback:

1. **Review** the specification
2. **Comment** on GitHub issues
3. **Test** with your server/client implementation
4. **Suggest** improvements or alternatives

Community input shapes the final design!

## Links

- **MCP Specification**: https://modelcontextprotocol.io/specification
- **RFC 2295**: https://tools.ietf.org/html/rfc2295 (Transparent Content Negotiation)
- **SEP Process**: https://modelcontextprotocol.io/community/sep-guidelines

## Status

- **Status**: Draft
- **Type**: Standards Track
- **Created**: February 22, 2026
- **Author**: Community (seeking sponsorship)

---

## Related Analysis Documents

This project includes supporting analysis:

- **RFC2295_ANALYSIS.md** - Deep dive into RFC 2295 concepts
- **MCP_ANALYSIS.md** - Current MCP architecture and capability model
- **MCP_PROPOSAL_GUIDE.md** - MCP SEP process and requirements

## License

This proposal is part of the MCP community process and follows MCP's licensing model.

---

**For questions or feedback**: Open an issue or discussion in this repository.

**To sponsor this SEP**: Contact the MCP maintainers.
