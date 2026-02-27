# Comparison: SEP-2090 Content Negotiation vs. SEP-2053 Server Variants

## Analysis

Both the Content Negotiation extension (`io.modelcontextprotocol/content-negotiation`) and SEP-2053 (Server Variants) address the same root problem: MCP servers cannot adapt to heterogeneous client types and are forced to return one-size-fits-all responses. Despite sharing this motivation — and arriving independently at overlapping vocabulary (`compact`/`verbose`, `interactive`/`autonomous-agent`) — the two proposals operate at different layers and make fundamentally different design choices.

Content Negotiation is a *session-scoped, client-driven* mechanism: clients declare structured feature tags once at `initialize`, and servers adapt response *content* (format, verbosity, structure) accordingly, with no per-request overhead and a hard constraint that tags never gate access. SEP-2053 is a *per-request, server-advertised* mechanism: servers publish named variants, clients select one per call via `_meta`, and servers may expose entirely different *tool surfaces* per variant.

The proposals are complementary rather than competing: SEP-2053 decides *which tools exist* for a given client profile; Content Negotiation decides *how results are formatted* within that surface. Implementing both allows a server to combine tool-catalog selection with fine-grained content adaptation.

---

This document compares this extension proposal with
[SEP-2053: Server Variants](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2053),
a related proposal by Sambhav Kothari (January 2026) that addresses the same
root problem from a different angle.

---

## Problem Statement: Where They Agree

Both proposals diagnose the same root cause: MCP servers cannot adapt to
different client types, forcing one-size-fits-all responses. Both use the
SEP-2133 Extensions framework with `capabilities.extensions` in `initialize`.
Both are fully backward-compatible.

---

## Core Design Philosophy: Where They Diverge

| Dimension | **Content Negotiation** (this extension) | **SEP-2053** (Server Variants) |
| --- | --- | --- |
| **Unit of negotiation** | Feature tags the client declares | Variant IDs the server advertises |
| **Who drives selection** | Client declares capabilities → server adapts | Server ranks variants → client picks |
| **What varies** | Response content (format, verbosity, structure) | Tool/resource surface (which tools exist, their schemas) |
| **Negotiation scope** | Session-scoped once at `initialize` | Per-request via `_meta` on every call |
| **Semantic grounding** | Tags tied to MCP protocol capabilities | Open-ended key-value hint strings |
| **Extension namespace** | `io.modelcontextprotocol/content-negotiation` | `io.modelcontextprotocol/server-variants` |

---

## Negotiation Mechanism

### Content Negotiation (this extension)

Client sends feature tags during `initialize`:

```json
{
  "capabilities": {
    "extensions": {
      "io.modelcontextprotocol/content-negotiation": {
        "version": "1.0",
        "features": ["agent", "!interactive", "format=json", "verbosity=compact"]
      }
    }
  }
}
```

Server reads those tags and **shapes every response** accordingly — same tools,
different content. No per-request overhead; decided once at handshake.

### Server Variants (SEP-2053)

Client sends open-ended hints during `initialize`:

```json
{
  "capabilities": {
    "extensions": {
      "io.modelcontextprotocol/server-variants": {
        "variantHints": {
          "description": "Claude-powered autonomous coding agent.",
          "hints": {
            "modelFamily": ["anthropic", "openai"],
            "useCase": ["autonomous-agent", "planning"]
          }
        }
      }
    }
  }
}
```

Server returns a ranked list of named variants. Client selects one and sends it
in `_meta` on **every request**:

```json
{
  "method": "tools/list",
  "params": {
    "_meta": { "io.modelcontextprotocol/server-variant": "compact" }
  }
}
```

Server can return **entirely different tool sets** per variant.

---

## Key Differentiators

### 1. Content vs. Surface

This extension shapes *content* (markdown vs. JSON, verbose vs. compact) while
keeping the tool surface identical. SEP-2053 shapes the *tool surface itself*
(different tool catalogs, different schemas per variant) — it is closer to
server "modes" than content negotiation.

### 2. Semantic Precision vs. Flexibility

Feature tags map to MCP protocol capabilities (`sampling`, `elicitation`,
`roots`, `tasks`) — they have well-defined semantics grounded in what the
client can actually do. SEP-2053's hints are open-ended key-value pairs
(`modelFamily`, `useCase`, vendor-namespaced keys) — more flexible but less
structured, requiring heuristic or LLM-based matching on the server side.

### 3. Session-Scoped vs. Per-Request Overhead

This extension is negotiated once; zero per-request overhead. SEP-2053 requires
sending `_meta` on every request plus significant additional complexity:
variant-scoped cursor namespacing, subscription binding per variant, and
`list_changed` notifications scoped to individual variants.

### 4. Security Model

This extension enforces that feature tags control *what is sent* (content
shape), never *whether to send it* (auth/access control). This is a hard
design constraint.

SEP-2053 blurs this boundary: variants like `"security-readonly"` and
`accessLevel` hint values (`"readonly"`, `"automation"`) use variant selection
as a de-facto access control mechanism. Relying on variant identity to gate
capabilities can create authorization gaps — a client that skips the `_meta`
field or guesses another variant ID may gain access to a broader tool surface
than intended.

### 5. Spec Complexity

SEP-2053 requires specifying:
- Variant-scoped pagination cursors (cursors from one variant must be rejected
  in another)
- Subscription binding (subscriptions lock to the variant active at subscribe
  time)
- Per-variant `list_changed` notifications
- Error recovery across variant switches

This extension has none of these concerns because content shaping is
session-scoped and does not affect which resources exist or how they are
addressed.

---

## What SEP-2053 Does Better

- **Tool-set switching**: Exposing entirely different tool catalogs to different
  agents (e.g., a `code-review` variant with 20 PR tools vs. an `automation`
  variant with 8 pipeline tools) is a real use case this extension does not
  address.
- **Per-request dynamism**: Switching variants mid-session (e.g., compact for
  execution, verbose for planning phases) is more powerful than session-scoped
  negotiation.
- **Named, inspectable configurations**: Variant IDs give clients something
  concrete to reason about, log, and expose in UIs.

## What This Extension Does Better

- **Protocol grounding**: Feature tags derive from MCP's own capability model;
  servers can make precise, verifiable content decisions.
- **Zero per-request overhead**: Session-once negotiation; no `_meta` plumbing
  on every call.
- **Simpler spec**: No cursor scoping, subscription binding, or per-variant
  notification routing.
- **Format and verbosity as first-class citizens**: `format=json|text|markdown`
  and `verbosity=compact|standard|verbose` are in the standard registry.
  SEP-2053 encodes these as ad-hoc hint strings with no canonical values.
- **Cleaner security model**: Tags cannot be used for access control by design.

---

## Overlap in Semantics

SEP-2053's `contextSize` hint values (`compact`, `standard`, `verbose`) and
`useCase` values (`interactive`, `autonomous-agent`) directly cover the same
semantic ground as this extension's `verbosity=compact|standard|verbose` and
`agent`/`!interactive` tags. Both proposals arrived at the same vocabulary
independently, which suggests these are the right axes for client-server
adaptation.

---

## Complementarity

These proposals address different layers and are not mutually exclusive:

- **SEP-2053** — decides *which tool surface* to expose (tool set selection)
- **This extension** — decides *how results are formatted* within that surface (content adaptation)

A client and server could implement both: use SEP-2053 variants to select the
appropriate tool catalog, then use content negotiation feature tags to shape how
tool results are formatted within that variant.

---

## Summary Table

| Criterion | Content Negotiation | SEP-2053 Server Variants |
| --- | --- | --- |
| Changes which tools exist | No | Yes |
| Changes response format/verbosity | Yes | Via ad-hoc hints only |
| Per-request `_meta` required | No | Yes |
| Spec complexity | Low | High |
| Security model clarity | Strong (no access control via tags) | Weaker (variants used for access gating) |
| Semantic grounding | MCP capability model | Open-ended hint strings |
| Complementary? | Yes | Yes |
