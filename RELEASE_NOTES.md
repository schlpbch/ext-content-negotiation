# Release Notes

## v0.9.3 — SEP-2053 Comparison Abstract
**Date**: February 27, 2026 | **Status**: Under Community Review

### Changes
- **Abstract added to SEP-2053 comparison**: `SEP2053_COMPARISON.md` now opens with a concise abstract summarising the shared motivation, the key design divergence, and the complementarity of the two proposals

---

## v0.9.2 — Documentation & Repo Guidance
**Date**: February 27, 2026 | **Status**: Under Community Review

### Changes
- **README compacted**: Reduced from 361 to ~130 lines; use cases and status collapsed into tables; removed redundant sections
- **CLAUDE.md added**: Claude Code guidance documenting file architecture, core concepts, and working conventions
- **Version badge updated** to v0.9.2 with corrected repository URL
- **Spec header updated**: Added `Version: v0.9.2` field to `ext-content-negotiation.md`
- **Tag & release target updated**: Tag `v0.9.2` moved to latest commit; release target set to `master`

---

## v0.9.1 — Extension Structure & Governance
**Date**: February 23, 2026 | **Status**: Under Community Review

### Changes
- **Restructured as Extensions Track**: SEP type changed from Standards Track → Extensions Track (per SEP-2133)
- **Fixed extension structure**: Client declares in `capabilities.extensions["io.modelcontextprotocol/content-negotiation"]`; server advertises with same field (empty object)
- **Removed** conflicting top-level capability declarations
- **Added** Extension Governance section to SEP
- **Updated** all initialization scenarios, code examples, and TypeScript schemas

---

## v0.9.0 — SEP Submission Release
**Date**: February 22, 2026 | **Status**: Under Community Review

Initial formal submission of the SEP for Transparent Content Negotiation in MCP.

### What's Included

**Core Specification** (`ext-content-negotiation.md`):
- Abstract, Motivation (4 use cases + workaround analysis), full Specification
- Rationale with 13-row RFC 2295 mapping table
- Backward Compatibility (zero breaking changes)
- Security Implications (5 risks with mitigations)
- Reference Implementation (TypeScript schemas and helpers)

**Supporting Analysis**:
- `RFC2295_ANALYSIS.md` — RFC 2295 concepts mapped to MCP
- `MCP_ANALYSIS.md` — MCP architecture and capability model analysis
- `MCP_PROPOSAL_GUIDE.md` — SEP process documentation

### Changes from v0.8
- Company-specific references removed (examples anonymized)
- Status badges added to README
- RFC 2295 links added throughout
- SEP issue references unified to [#2290](https://github.com/modelcontextprotocol/specification/issues/2290)

---

## Submission Status

- **Submission Issue**: [modelcontextprotocol/specification#2290](https://github.com/modelcontextprotocol/specification/issues/2290)
- ✅ Draft Phase: Community feedback and refinement (ongoing)
- ⏳ In-Review Phase: Awaiting core maintainer review and SEP number assignment
- ⏳ Final Phase: Approval and integration into MCP specification

## Files

```
ext-content-negotiation/
├── README.md                    # Overview and quick-start
├── ext-content-negotiation.md  # Complete SEP specification
├── RFC2295_ANALYSIS.md         # RFC 2295 analysis
├── MCP_ANALYSIS.md             # MCP architecture analysis
├── MCP_PROPOSAL_GUIDE.md       # Proposal process guide
├── CLAUDE.md                   # Claude Code guidance
├── SEP2053_COMPARISON.md       # Comparison with SEP-2053 Server Variants
└── RELEASE_NOTES.md            # This file
```
