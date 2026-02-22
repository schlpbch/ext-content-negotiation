# Release Notes - v0.9.0

**Date**: February 22, 2026
**Status**: Under Community Review
**Type**: Standards Track Proposal

---

## Overview

**Version 0.9.0** is the formal submission release of the Specification Enhancement Proposal (SEP) for Transparent Content Negotiation in the Model Context Protocol (MCP). This release contains the complete, verified specification ready for community discussion and formal review.

---

## What's Included

### 📋 Core Specification

**File**: `SEP-DRAFT-agent-content-negotiation.md` (1,210 lines)

A comprehensive formal proposal containing:

- **Abstract** — Problem statement and solution overview
- **Motivation** — 4 real-world use cases and analysis of existing workarounds
- **Specification** — Complete technical specification with:
  - New `ClientContentNegotiationCapability` interface
  - New `ServerContentNegotiationCapability` interface
  - Feature tag vocabulary and predicate syntax
  - 3 complete initialize scenarios (agent, human, legacy)
  - Tool response variants with JSON examples
  - Resource content negotiation examples
  - Prompt template adaptation patterns
- **Rationale** — Design decisions with:
  - 13-row RFC 2295 mapping table (adopted, adapted, rejected, deferred)
  - Alternatives considered and analysis
  - Why session-scoped over per-request negotiation
  - Why capability-driven over generic labels
- **Backward Compatibility** — Zero breaking changes with migration path
- **Security Implications** — 5 identified risks with mitigations:
  - Feature tag spoofing (limited impact)
  - Tag injection and validation
  - Semantic ambiguity handling
  - Auth/access control separation
  - Privacy considerations
- **Reference Implementation** — TypeScript schemas and helpers:
  - Interface definitions for client and server
  - Feature parsing helper functions
  - Complete weather tool example
  - Tool response variant examples
- **Additional Sections**:
  - Performance considerations
  - Testing strategy with 4 test cases
  - Open questions for community feedback

### 📚 Documentation

**README.md** (265 lines)
- Problem statement and solution overview
- 4 real-world use cases
- Feature tag registry with reference table
- Key features checklist
- Quick start code examples
- Security & privacy guidelines
- How to review guide for MCP maintainers
- Status badges and submission links

**Supporting Analysis Documents**:
- `RFC2295_ANALYSIS.md` — Deep dive into RFC 2295 concepts with MCP application section
- `MCP_ANALYSIS.md` — MCP architecture and capability model analysis with content negotiation application
- `MCP_PROPOSAL_GUIDE.md` — MCP SEP process documentation with real-world example

---

## Key Features (v1.0)

### Feature Tag Registry

| Category | Tags | Purpose |
|---|---|---|
| Client Type | `agent`, `human` | Content format preference |
| Capability Awareness | `mcp-capable` | Direct tool/resource name references |
| MCP Capabilities | `sampling`, `elicitation`, `roots`, `tasks` | Reasoning hints, instruction support |
| Interactivity | `interactive`, `!interactive` | UI presentation capability |
| Output Format | `format=json\|text\|markdown` | Structured vs prose content |
| Response Length | `verbosity=compact\|standard\|verbose` | Explanation detail level |
| Custom | `x-*` | Vendor-specific extensions |

### Design Principles

- **RFC 2295-Inspired** — Borrows transparent content negotiation concepts from HTTP
- **Session-Scoped** — Declared once at `initialize`, not per-request
- **Capability-Driven** — Feature tags mirror client's actual MCP capabilities
- **Flexible Predicates** — Supports presence, negation, and equality syntax
- **Fully Backward Compatible** — Zero breaking changes, completely optional
- **Secure** — Feature tags for content shaping only, never for auth/access control

---

## Use Cases

### 1. Journey Service
Serve connection itineraries in JSON format for agents, readable markdown for humans—one tool, negotiated format.

### 2. Geospatial/Mapping Service
Return GeoJSON coordinates for agents, annotated markdown descriptions for humans from the same resource.

### 3. Multi-Agent Orchestration
Orchestrator reads each agent's declared capabilities and servers adapt responses for optimal agent processing.

### 4. Environmental Services
Weather, air quality, and environmental data served as structured JSON for agents, narrative prose for humans.

---

## What Changed from Previous Versions

### From v0.8 → v0.9.0

- ✅ Company-specific references removed (all examples anonymized)
- ✅ Status badges added to README
- ✅ RFC 2295 links added throughout documentation
- ✅ All SEP issue references unified to #2290
- ✅ Official RFC Editor URL for RFC 2295
- ✅ Comprehensive GitHub release notes

### Original Features (Retained)

- ✅ Complete 1,210-line SEP specification
- ✅ 7 required SEP sections verified
- ✅ Feature tag vocabulary with 14 standard tags
- ✅ 3 initialize scenarios documented
- ✅ 2+ tool response pairs (agent vs human)
- ✅ RFC 2295 mapping analysis (13 rows)
- ✅ Security analysis with 5 risks
- ✅ TypeScript reference implementation
- ✅ Correct SEP header format

---

## How to Use This Release

### For MCP Maintainers

1. **Review the Specification**
   - Start with the Abstract (2 min)
   - Read the Motivation section (10 min)
   - Examine the Specification section (15 min)
   - Check the Rationale section (5 min)

2. **Assess Implementation Feasibility**
   - Review the Reference Implementation section
   - Check TypeScript schema additions
   - Validate helper function patterns

3. **Evaluate Security & Compatibility**
   - Review the Security Implications section
   - Verify backward compatibility analysis
   - Check migration path

### For Community Members

1. **Read the README.md** for overview and quick-start examples
2. **Review the full SEP** in `SEP-DRAFT-agent-content-negotiation.md`
3. **Provide feedback** on [GitHub Issue #2290](https://github.com/modelcontextprotocol/specification/issues/2290)
4. **Test** with your own MCP server/client implementations

### For Implementers

Use the **Quick Start** section in README.md:

**Server Side**:
```typescript
const features = client.capabilities.contentNegotiation?.features || [];
const isAgent = features.includes("agent");
const wantsJson = features.includes("format=json");

if (isAgent && wantsJson) {
  return { structuredContent: {...} };  // JSON
} else {
  return { content: [{type: "text", text: "..."}] };  // Markdown
}
```

**Client Side**:
```typescript
const capabilities = {
  sampling: {},
  contentNegotiation: {
    version: "1.0",
    features: ["agent", "sampling", "format=json", "verbosity=compact"]
  }
};
```

---

## Submission Status

- **Status**: Submitted for Review
- **Submission Date**: February 22, 2026
- **Submission Issue**: [modelcontextprotocol/specification#2290](https://github.com/modelcontextprotocol/specification/issues/2290)
- **Timeline**:
  - ✅ **Draft Phase**: Community feedback and refinement (ongoing)
  - ⏳ **In-Review Phase**: Awaiting core maintainer review and SEP number assignment
  - ⏳ **Final Phase**: Approval and integration into MCP specification

---

## Getting Involved

### Review the Proposal
1. Read [README.md](README.md) for context
2. Review [SEP-DRAFT-agent-content-negotiation.md](SEP-DRAFT-agent-content-negotiation.md)
3. Check [RFC2295_ANALYSIS.md](RFC2295_ANALYSIS.md) for RFC background

### Provide Feedback
- Comment on [GitHub Issue #2290](https://github.com/modelcontextprotocol/specification/issues/2290)
- Suggest improvements or alternatives
- Report implementation experience

### Test & Implement
- Build a proof-of-concept implementation
- Test with your MCP server/client
- Share results and lessons learned

---

## Technical Details

### Dependencies
- No runtime dependencies
- TypeScript 4.5+ for type definitions
- MCP specification v1.x

### Compatibility
- **Backward Compatible**: ✅ Yes, fully optional
- **Breaking Changes**: ❌ None
- **Migration Path**: Available in Backward Compatibility section

### Platform Support
- Browser-based clients: ✅ Supported
- Node.js servers: ✅ Supported
- Any MCP-compatible implementation: ✅ Supported

---

## Files in This Release

```
agentic-content-negotiation/
├── README.md                                   # Main documentation
├── SEP-DRAFT-agent-content-negotiation.md     # Complete specification
├── RFC2295_ANALYSIS.md                        # RFC 2295 analysis
├── MCP_ANALYSIS.md                            # MCP architecture analysis
├── MCP_PROPOSAL_GUIDE.md                      # Proposal process guide
└── RELEASE_NOTES.md                           # This file
```

---

## Questions?

- **About the proposal?** Open a discussion on [GitHub Issue #2290](https://github.com/modelcontextprotocol/specification/issues/2290)
- **About implementation?** See the Reference Implementation section in the SEP
- **About RFC 2295?** Read [RFC2295_ANALYSIS.md](RFC2295_ANALYSIS.md)

---

## Credits

- **Concept**: Inspired by RFC 2295 (Transparent Content Negotiation for HTTP)
- **Adapted for MCP**: By the community
- **Proposal Tracking**: [modelcontextprotocol/specification#2290](https://github.com/modelcontextprotocol/specification/issues/2290)

---

**Version**: 0.9.0
**Released**: February 22, 2026
**Status**: Under Community Review
**Type**: Standards Track Enhancement Proposal
