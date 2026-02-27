# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A **Standards Enhancement Proposal (SEP)** for the Model Context Protocol (MCP), proposing a content negotiation extension (`io.modelcontextprotocol/content-negotiation`). This is a **pure documentation repository** — there are no build tools, tests, package.json, or runnable code.

The proposal is tracked at [modelcontextprotocol/specification#2290](https://github.com/modelcontextprotocol/specification/issues/2290) and targets the [SEP-2133: Extensions](https://modelcontextprotocol.io/community/seps/2133-extensions) framework.

## Document Architecture

| File | Role |
| --- | --- |
| `ext-content-negotiation.md` | The canonical SEP specification (~1,200 lines). Primary document. |
| `README.md` | Overview and quick-start for community reviewers. |
| `RELEASE_NOTES.md` | Release history and submission status. |
| `RFC2295_ANALYSIS.md` | Source material: how RFC 2295 maps to MCP. |
| `MCP_ANALYSIS.md` | MCP architecture analysis used to ground the proposal. |
| `MCP_PROPOSAL_GUIDE.md` | SEP process documentation. |

## Core Concepts

**The extension**: Clients declare feature tags during `initialize`; servers adapt response format accordingly. Negotiation is session-scoped (once per connection), not per-request.

**Feature tags** are the unit of negotiation. Standard tags: `agent`, `human`, `mcp-capable`, `interactive`, `!interactive`, `sampling`, `elicitation`, `roots`, `tasks`, `format=json|text|markdown`, `verbosity=compact|standard|verbose`, `x-*` (vendor).

**Capability path** in the MCP `initialize` message:
```
capabilities.extensions["io.modelcontextprotocol/content-negotiation"].features
```

**Key design constraint**: Feature tags control *what* the server sends (content shape), never *whether* to send it (auth/access). This is central to the security model.

## Working in This Repo

- Changes to `ext-content-negotiation.md` are the most significant — it is the formal specification that will be submitted to MCP maintainers.
- `README.md` is the community-facing entry point; keep it concise and aligned with the spec.
- The analysis documents (`RFC2295_ANALYSIS.md`, `MCP_ANALYSIS.md`) are reference/background material — edit only to fix errors, not to add new design decisions.
- Code snippets in the spec are TypeScript illustrating the MCP SDK interface shape, not runnable code.
