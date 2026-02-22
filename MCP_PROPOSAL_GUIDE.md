# How to Create a Specification Proposal for the Model Context Protocol (MCP)

**Document Date**: February 22, 2026
**Target Audience**: Protocol contributors, specification authors, MCP implementors
**Document Type**: Contributing Guide & Process Analysis

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Overview of the SEP Process](#overview-of-the-sep-process)
3. [Governance Structure](#governance-structure)
4. [Prerequisites & Preparation](#prerequisites--preparation)
5. [SEP Types & Categories](#sep-types--categories)
6. [Step-by-Step Proposal Process](#step-by-step-proposal-process)
7. [SEP Template & Structure](#sep-template--structure)
8. [Writing Effective Proposals](#writing-effective-proposals)
9. [Review & Feedback Process](#review--feedback-process)
10. [Reference Implementation Requirements](#reference-implementation-requirements)
11. [Common Mistakes to Avoid](#common-mistakes-to-avoid)
12. [Real-World Examples](#real-world-examples)
13. [Technical Contribution Requirements](#technical-contribution-requirements)

---

## Executive Summary

The Model Context Protocol uses a **Specification Enhancement Proposal (SEP)** system to manage changes to the protocol specification, similar to Python Enhancement Proposals (PEPs) or Rust RFCs.

**Key Facts**:
- SEP process is formalized and documented
- Community-driven with transparent governance
- Requires motivation, specification, rationale, and reference implementation
- Three SEP types: Standards Track, Informational, Process
- Built on principle of "rough consensus and running code"
- Final approval by Core Maintainers and Lead Maintainers

**Typical Timeline**: 2-8 weeks from proposal to acceptance (depending on scope)

**Success Rate**: Varies based on proposal quality, community alignment, and author involvement

---

## Overview of the SEP Process

### What is a SEP?

A **Specification Enhancement Proposal** is a design document providing information to the MCP community, or describing a new feature/change to MCP protocol or processes.

**Purpose**:
- Propose new protocol features (primitives, message types)
- Suggest changes to existing functionality
- Document governance and process improvements
- Share important information with the community

**Process Philosophy**: "Rough consensus and running code"
- Community input shapes the design
- Reference implementation validates feasibility
- Final decision by Core Maintainers

### SEP Lifecycle

```
Idea Discussion → Draft SEP → Community Review → Reference Impl →
    ↓
Final Review → Accepted/Rejected
    ↓
 [If Accepted] → Documentation → Merged to Specification
```

**Typical States**:
- **Draft**: Initial proposal, gathering feedback
- **In-Review**: Undergoing community review, possibly with reference implementation
- **Accepted**: Approved by Core Maintainers, implementation in progress
- **Final**: Complete implementation, merged to spec
- **Rejected**: Community/maintainers decided against proposal
- **Superseded**: Newer SEP replaces this one
- **Withdrawn**: Author withdrew the proposal
- **Dormant**: Inactive proposal, may be revisited

---

## Governance Structure

### Decision-Making Hierarchy

```
┌─────────────────────────────────────────┐
│ Lead Maintainers (2)                    │
│ - Justin Spahr-Summers                  │
│ - David Soria Parra                     │
│ - Can veto any decision                 │
│ - Appoint/remove Core Maintainers       │
└─────────────────────────────────────────┘
           ↑
┌─────────────────────────────────────────┐
│ Core Maintainers (5-10)                 │
│ - Deep MCP understanding required       │
│ - Responsible for protocol evolution    │
│ - Meet bi-weekly for decisions          │
│ - Vote on major proposals (majority)    │
│ - Can veto maintainer decisions         │
└─────────────────────────────────────────┘
           ↑
┌─────────────────────────────────────────┐
│ Maintainers                             │
│ - Component-specific responsibility     │
│ - Write/admin access to repos           │
│ - Appointed by Core Maintainers         │
│ - Review PRs in their domains           │
└─────────────────────────────────────────┘
           ↑
┌─────────────────────────────────────────┐
│ Contributors                            │
│ - Anyone filing issues, PRs, comments   │
│ - Can propose SEPs                      │
│ - No formal membership required         │
└─────────────────────────────────────────┘
```

### Important Notes

- **Individual, not Corporate**: Governance tied to individuals, not companies
  - Prevents corporate capture of project direction
  - Maintains continuity across employer changes

- **Transparent Process**: All discussions documented and public
  - GitHub issues for coordination
  - Pull requests for proposals
  - Bi-weekly Core Maintainer meetings (notes public)

- **Accessibility**: Everyone can participate in discussions
  - No voting rights required to provide input
  - Community consensus matters to decision-makers

---

## Prerequisites & Preparation

### Before You Write a SEP

#### 1. Validate Your Idea
```
Do this BEFORE investing in a full SEP:
- [ ] Search existing SEPs for similar proposals
- [ ] Check GitHub issues for related discussions
- [ ] Review recent protocol versions for related features
- [ ] Discuss idea in community channels (Discord, GitHub discussions)
```

**Community Validation Steps**:
1. Post in MCP community discussions
2. Share use case with maintainers
3. Get initial feedback on feasibility
4. Gauge community interest level

#### 2. Understand Current Architecture
- Read the full specification
- Study existing SEPs
- Understand capability negotiation
- Review core primitives (prompts, resources, tools)

#### 3. Identify Your SEP Type
- **Standards Track**: Core protocol changes
- **Informational**: Documentation, guidance (not protocol changes)
- **Process**: Governance, workflow changes

#### 4. Build Your Use Case
Essential for Standards Track SEPs:
- Real-world problem you're solving
- Current workarounds and limitations
- Community benefit of your proposal
- Timeline/scalability impact

### Required Knowledge

**Protocol Understanding**:
- JSON-RPC 2.0 message format
- Client-host-server architecture
- Capability negotiation system
- Message lifecycle and state management
- Existing primitives and their semantics

**Implementation Understanding**:
- How servers expose capabilities
- How clients invoke requests
- Error handling patterns
- Backward compatibility considerations

**Process Understanding**:
- SEP format and template
- GitHub workflow (fork, branch, PR)
- TypeScript schema conventions
- Documentation format (MDX)

### Development Environment Setup

```bash
# Clone the repository
git clone https://github.com/your-username/modelcontextprotocol.git
cd modelcontextprotocol

# Install dependencies
nvm install  # install correct Node version
npm install  # install dependencies

# Create a feature branch
git checkout -b sep/your-feature-name
```

**Required Tools**:
- Node.js 20+ (managed by nvm)
- TypeScript
- Git
- Text editor (VS Code recommended)
- (Optional) Hugo for blog changes

---

## SEP Types & Categories

### 1. Standards Track SEPs

**Purpose**: Propose changes to core protocol specification

**Scope**:
- New protocol primitives
- New message types or endpoints
- Changes to existing capabilities
- Breaking changes to protocol behavior
- Major architectural changes

**Requirements**:
- Comprehensive motivation section
- Detailed specification with examples
- Rationale explaining design decisions
- Backward compatibility analysis (if applicable)
- Reference implementation (before Final status)
- Security implications assessment

**Example SEPs**:
- SEP-1686: Tasks (long-running operations)
- SEP-1330: Enum schema improvements
- SEP-1699: SSE polling via server-side disconnect

### 2. Informational SEPs

**Purpose**: Provide information to community, not propose protocol changes

**Scope**:
- Best practices and guidelines
- Implementation recommendations
- Conceptual explanations
- Process documentation
- Historical records

**Requirements**:
- Clear explanation of topic
- Practical examples or guidance
- No requirement for implementation
- Community benefit clearly articulated

**Example SEPs**:
- Governance documentation
- Communication guidelines
- Implementation tips

### 3. Process SEPs

**Purpose**: Propose changes to project governance or processes

**Scope**:
- SEP process changes
- Governance structure modifications
- Contribution guidelines
- Release procedures
- Community organization

**Requirements**:
- Clear rationale for change
- Step-by-step procedures
- Role and responsibility definitions
- Timeline if applicable
- Implementation plan

**Example SEPs**:
- SEP-932: Governance structure
- SEP-1850: PR-based SEP workflow
- SEP-2149: Working group charter

---

## Step-by-Step Proposal Process

### Phase 1: Preparation (Week 1)

#### Step 1: Get SEP Number
Before writing, request a SEP number:
1. Open a GitHub issue in the specification repository
2. Title: "SEP Request: [Your Feature]"
3. Include brief description of your proposal
4. Maintainers will assign number and close issue

**Example Issue**:
```
Title: SEP Request: Enhanced Sampling Support

Description:
I'm planning to propose enhancements to the sampling protocol
to support additional LLM models and configuration options.
Expected scope: 3-4 weeks to implementation.

Type: Standards Track
```

#### Step 2: Validate Scope
- Confirm your proposal aligns with maintainer roadmap
- Discuss timeline expectations
- Clarify any questions about existing architecture

### Phase 2: Draft & Community Discussion (Week 2-3)

#### Step 3: Write Initial Draft
Use the SEP template (see Section 7)

**Draft Structure**:
```
- Status: Draft
- Type: [Standards Track/Informational/Process]
- Created: YYYY-MM-DD
- Author(s): Your Name <email> (@github)
- Abstract: 200-word summary
- Motivation: Problem statement
- Specification: Technical details
- Rationale: Design decisions
- Reference Implementation: Link or description
- [Additional sections as needed]
```

#### Step 4: Gather Initial Feedback
- Post draft in GitHub discussion
- Share with community channels
- Get input from potential users
- Refine based on feedback

**Where to Share**:
- GitHub Discussions in specification repo
- MCP Community Discord
- Twitter/X #MCP hashtag
- Email to maintainers

### Phase 3: Formal Submission (Week 3-4)

#### Step 5: Create Pull Request
1. Fork the specification repository
2. Create branch: `sep/[number]-[slug]`
3. Create file: `seps/[number]-[slug].md`
4. Add SEP markdown file to seps/ directory
5. Push to your fork

**File Naming Convention**:
```
seps/1686-tasks.md
seps/1699-sse-polling.md
seps/932-governance.md
```

**PR Format**:
```
Title: SEP-XXXX: [Your Feature Title]

Description:
- Type: Standards Track
- Status: Draft
- Closes: #XXXX (related issue if any)

[Optional: Brief description of changes]
```

#### Step 6: Link SEP to PR
Update SEP header with PR reference:
```markdown
- PR: https://github.com/modelcontextprotocol/specification/pull/{NUMBER}
```

### Phase 4: Community Review (Week 4-6)

#### Step 7: Respond to Feedback
- Maintainers and community comment on PR
- Address concerns with:
  - Clarifications in spec
  - Design changes if needed
  - Additional examples
  - Evidence of feasibility

**Review Considerations**:
- Technical soundness
- Backward compatibility
- Security implications
- Community alignment
- Reference implementation quality

#### Step 8: Iterate on Specification
- Update SEP based on feedback
- Commit changes to PR branch
- Re-request review if major changes made

**Don't Rebase**: Push new commits for visibility of changes

### Phase 5: Reference Implementation (Week 5-7)

#### Step 9: Develop Reference Implementation
Before SEP can reach Final status:

**Requirements**:
- Working code demonstrating the proposal
- Can be prototype or production-ready
- Should handle happy path and errors
- Include tests/examples

**Where to Put It**:
- TypeScript schema changes: `schema/draft/schema.ts`
- SDK examples: linked from PR or external repo
- Test code: new test files

**Process**:
1. Implement in schema/draft if applicable
2. Update generated JSON schema: `npm run generate:schema`
3. Add documentation: `docs/specification/draft/[feature]/`
4. Create tests or examples
5. Link from SEP: "Reference Implementation" section

#### Step 10: Validate Implementation
- Code review by maintainers
- Type checking: `npm run check:schema:ts`
- Schema validation: `npm run check:schema:json`
- Documentation build: `npm run generate:schema`
- Link verification: `npm run check:docs:links`

### Phase 6: Final Review (Week 7-8)

#### Step 11: Final Core Maintainer Review
- Core Maintainers review SEP + implementation
- Bi-weekly meeting (check schedule)
- Discuss any outstanding concerns
- Vote on acceptance

**Acceptance Criteria**:
- Specification is technically sound
- Implementation validates feasibility
- Community consensus (no major objections)
- Backward compatibility addressed
- Security reviewed

#### Step 12: Update Status
Once accepted, update SEP header:
```markdown
- Status: Accepted
- PR: [Updated with final PR number if changed]
```

### Phase 7: Finalization & Merge (Week 8+)

#### Step 13: Merge to Main Specification
1. PR approved and merged to main branch
2. Changes appear in current spec version
3. Update changelog/release notes
4. Announce in community

#### Step 14: Update Protocol Version
- May require new protocol version number
- Documentation updated
- SDKs updated to support new feature

---

## SEP Template & Structure

### Full SEP Template

```markdown
# SEP-{NUMBER}: {Title}

- **Status**: Draft | In-Review | Accepted | Rejected | Withdrawn | Final | Superseded | Dormant
- **Type**: Standards Track | Informational | Process
- **Created**: YYYY-MM-DD
- **Author(s)**: Name <email> (@github-username)
- **Sponsor**: @github-username (or "None" if seeking)
- **PR**: https://github.com/modelcontextprotocol/specification/pull/{NUMBER}

## Abstract

[Brief ~200 word technical summary. Should allow readers to quickly understand proposal.]

## Motivation

[Why is this change needed? Why is current spec inadequate?
Motivation is CRITICAL. SEPs without sufficient motivation may be rejected outright.]

## Specification

[Detailed technical specification. Sufficient detail for interoperable implementations.]

For Protocol changes, include:
- New message formats or data structures
- Endpoints or methods
- Behavioral requirements
- Error handling

For Process changes, include:
- Step-by-step procedures
- Roles and responsibilities
- Timelines or milestones

## Rationale

[Why these design decisions? Include:
- Alternate designs considered
- Why proposed approach chosen
- Related work or prior art
- Objections/concerns raised
- Evidence of community consensus]

## Backward Compatibility

[REQUIRED for incompatible changes. Describe:
- What existing functionality breaks
- Severity and scope
- Proposed transition path
- Migration for existing implementations

If no concerns: "No backward compatibility concerns."]

## Security Implications

[Describe security concerns related to proposal:
- New attack surfaces
- Privacy considerations
- Auth/authz changes
- Data validation requirements

If none: "No security implications."]

## Reference Implementation

[Link to or describe reference implementation.
REQUIRED before Final status.

Include:
- Links to prototype code or PRs
- Pointers to example usage
- Test results or validation]

## Additional Sections (Optional)

### Performance Implications
How does this affect performance, scalability, resource usage?

### Testing Plan
How will proposal be tested? What test cases?

### Alternatives Considered
Detailed discussion of rejected alternatives and why.

### Open Questions
Unresolved issues needing community input.

### Acknowledgments
Credit to contributors and reviewers.
```

### Template Customization

**The template is flexible** - adapt based on proposal needs:

**Standards Track SEP**:
- Always include: Abstract, Motivation, Specification, Rationale, Backward Compatibility, Security, Reference Implementation
- Usually include: Performance, Testing Plan, Alternatives

**Informational SEP**:
- Can omit: Backward Compatibility (if not applicable)
- Focus on: Clear explanation, practical guidance, examples

**Process SEP**:
- Can omit: Backward Compatibility (unless applicable)
- Include: Step-by-step procedures, roles, timeline

---

## Writing Effective Proposals

### Section: Abstract

**Purpose**: Quick overview allowing readers to understand proposal in 5 seconds

**Length**: ~200 words (soft limit)

**Content**:
- What problem does it solve?
- What is the proposed solution?
- Key technical approach

**Example** (from SEP-1686):
> "This SEP improves support for task-based workflows in the Model Context Protocol (MCP). It introduces both the task primitive and the associated task ID, which can be used to query the state and results of a task, up to a server-defined duration after the task has completed. This primitive is designed to augment other requests (such as tool calls) to enable call-now, fetch-later execution patterns across all requests for servers that support this primitive."

### Section: Motivation

**Purpose**: Convince readers your proposal addresses a real problem

**Critical for Acceptance**: Weak motivation = automatic rejection

**Content**:
- Current limitations/pain points
- Real-world use cases
- Community demand
- Why existing workarounds are insufficient

**Strong Motivation Includes**:
- Specific customer use cases (anonymized if needed)
- Data showing demand (e.g., GitHub issues, survey results)
- Limitations of current approach (with examples)
- Impact if problem not solved
- Timeline urgency if relevant

**Example** (from SEP-1686):
```
Current MCP has no way to:
- Explicitly request status of tool call
- Retrieve result after completion

Workarounds (inadequate):
- Server splits tool into three separate tools (start/get-status/get-result)
- Requires agent polling (expensive, unreliable)
- No way for host to take ownership

Real use cases failing today:
1. Healthcare: Drug discovery workflows (hours long)
2. Enterprise: SDLC automation
3. Code migration: Multi-hour transformations
4. Test execution: Concurrent test monitoring
5. Deep research: Multi-agent coordination
6. Agent-to-agent: Parallel processing

Impact: Customers leaving MCP for existing platforms
```

### Section: Specification

**Purpose**: Technical detail sufficient for independent implementations

**Critical Details**:
- New message formats
- Protocol state changes
- Behavior requirements
- Error handling
- Examples of actual messages

**Structure**:
1. Overview of changes
2. Data structures (JSON examples)
3. Message types (new/modified)
4. Behavior requirements
5. Error conditions
6. Concrete examples

**Style**:
- Use "MUST", "SHOULD", "MAY" (RFC 2119 keywords)
- Include actual JSON message examples
- Show state transitions if applicable
- Explain new fields in detail

**Example Structure**:
```markdown
## Specification

### Overview
This change introduces the Task primitive for...

### Data Structures

#### Task Definition
```json
{
  "id": "task-123",
  "state": "running|completed|failed",
  "result": { ... },
  "created": "2025-02-22T10:00:00Z"
}
```

### Message Types

#### New: tasks/status
Retrieve current status of a task.

**Request**:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tasks/status",
  "params": {
    "taskId": "task-123"
  }
}
```

**Response**:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "taskId": "task-123",
    "state": "running",
    "progress": 65
  }
}
```

### Requirements
- Task IDs MUST be unique per server
- Servers MUST support tasks/status if tasks capability enabled
- Tasks MUST be retained for at least N minutes after completion
- ...
```

### Section: Rationale

**Purpose**: Explain why design decisions were made this way

**Content**:
- Alternate approaches considered (with pros/cons)
- Why this approach chosen
- Related work or prior art
- Objections raised and responses
- Evidence of consensus

**Format**:
```markdown
## Rationale

### Why not approach X?
[Explanation of why X was rejected]
- Limitation 1
- Limitation 2
- Better alternative: Y

### Why this approach?
[Why chosen approach is superior]
- Advantage 1 (supported by evidence)
- Advantage 2
- Community preference (link to discussion)

### Prior art
[Related systems doing this]
- Kubernetes Jobs API: ...
- AWS Step Functions: ...
- GitHub Actions: ...

### Objections and responses
[From community discussion]
- Objection 1: "This is too complex"
  Response: Complexity justified by use cases...
```

### Section: Backward Compatibility

**Required for Standards Track SEPs with any incompatibilities**

**Content**:
- What breaks?
- Who is affected?
- Transition plan
- Migration path

**Example**:
```markdown
## Backward Compatibility

### Breaking Changes
- Task capability added to capability negotiation
- Servers must declare task support
- Existing servers without task support unaffected

### Transition Period
- Version 2025-11-25: Tasks introduced as optional
- Version 2026-06-01: Tasks available but optional
- Version 2027-01-01: Could be mandatory (not planned)

### Migration Path
1. Implement task capability in schema
2. Declare tasks capability in initialize response
3. Implement tasks/status and tasks/result endpoints
4. Update to new protocol version
```

### Writing Quality Standards

**Clarity**:
- Technical writing (not marketing)
- Precise terminology
- Spell check: `npm run check:docs:format`
- Examples in actual JSON

**Completeness**:
- All new concepts defined
- All edge cases addressed
- Error conditions documented
- Examples for common patterns

**Evidence**:
- Real use cases, not hypothetical
- Links to GitHub issues/discussions
- References to existing implementations
- Data supporting motivation

---

## Review & Feedback Process

### What to Expect

**Community Review Period**: 2-4 weeks typically

**Review Participants**:
- Core Maintainers (final decision)
- Other contributors (feedback)
- SDK maintainers (implementation concerns)
- Community members (use case validation)

### Types of Feedback

#### Technical Feedback
- Design concerns
- Implementation issues
- Message format improvements
- Schema changes

**Response**:
- Address with spec updates
- Explain decisions if no change

#### Use Case Feedback
- "This doesn't solve my problem because..."
- "Here's a better approach..."
- "This conflicts with my use case..."

**Response**:
- Update motivation section
- Refine specification if applicable
- Document why certain approaches chosen

#### Process Feedback
- "This should go through working group X"
- "This needs security review"
- "This affects SDKs"

**Response**:
- Coordinate with relevant parties
- Add to review checklist
- Schedule appropriate reviews

### Responding to Criticism

**Best Practices**:
1. **Assume good faith** - Reviewers want good protocol
2. **Address substantively** - Don't dismiss concerns
3. **Explain thinking** - Why you disagree (if you do)
4. **Be willing to revise** - Good feedback improves proposal
5. **Request clarification** - If you don't understand concern
6. **Track all changes** - Commit messages explaining updates

**Template Response**:
```
Thank you for the feedback, @reviewer!

Regarding your concern about X:

Current spec handles this by Y.

If you're suggesting Z as alternative:
- Pro: Better for use case A
- Con: Breaks existing approach B
- My preference: Y because [reason]

What do you think? Happy to revise if you disagree.
```

### When to Push Back

**Legitimate reasons to disagree with feedback**:
- Feedback contradicts spec requirements
- Concern already addressed in another section
- Alternative approach creates worse problems
- Community consensus against suggestion

**How to push back professionally**:
```
I appreciate the suggestion, but I respectfully disagree because:

1. [Technical reason with evidence]
2. [Spec already handles this via X]
3. [Community feedback supported this approach]

I'm happy to discuss further, or we can note this as an open question
for Core Maintainers to decide.
```

---

## Reference Implementation Requirements

### Why Reference Implementation is Critical

**Principle**: "Rough consensus and running code"

**Purpose**:
- Validates that proposal is technically feasible
- Reveals gaps in specification
- Provides example for other implementations
- Demonstrates actual behavior

**When Required**:
- MUST have before "Final" status
- Not required for "Accepted" status
- Strongly recommended for Standards Track

### Implementation Requirements

**Quality Bar**:
- Handles happy path completely
- Includes error handling
- Demonstrates interoperability (if applicable)
- Includes tests or examples
- Production-quality code (not prototype)

**Scope**:
- Must implement entire feature in proposal
- Can be in part of MCP (e.g., just schema changes)
- Can be separate repository (linked from SEP)
- Can be in draft schema during review

### Implementation Locations

**Option 1: Schema Changes Only**
```
Modify: schema/draft/schema.ts
Run: npm run generate:schema
Result: Automatic JSON Schema + docs generation
```

**Option 2: Documentation Examples**
```
Add: docs/specification/draft/[feature]/example.mdx
Include: Working code samples, message examples
```

**Option 3: External Reference Implementation**
```
Create: Separate repository with full implementation
Link: From SEP "Reference Implementation" section
Validate: Through automated testing
```

### Implementation Validation

**Before marking complete**:
1. Type check passes: `npm run check:schema:ts`
2. Schema generates: `npm run generate:schema`
3. Docs build: `npm run serve:docs`
4. No lint errors: `npm run check:schema`
5. Tests pass: `npm run check:schema:examples`

**Evidence to Include**:
- Links to code commits
- Screenshots of tests passing
- Example output
- Interop testing results (if multi-party)

---

## Common Mistakes to Avoid

### Mistake 1: Insufficient Motivation

**Problem**: "This would be nice to have" without showing real need

**Impact**: Rejection or indefinite stalling

**Fix**:
- ✓ Real customer use cases
- ✓ GitHub issues showing demand
- ✓ Data on pain point frequency
- ✓ Impact if problem not solved

### Mistake 2: Too Much Implementation Detail, Too Little Specification

**Problem**: Big implementation PR without clear spec document

**Impact**: Hard to review, unclear requirements

**Fix**:
- ✓ Spec document first
- ✓ Then implementation
- ✓ Spec should be readable without code

### Mistake 3: Backward Incompatibility Not Addressed

**Problem**: Breaking changes without migration plan

**Impact**: Rejection, or requirements to redesign

**Fix**:
- ✓ Explicitly state what breaks
- ✓ Provide transition period
- ✓ Offer migration path
- ✓ Justify why breaking change necessary

### Mistake 4: Scope Creep

**Problem**: Combining multiple features in one SEP

**Impact**: Too complex to review, harder to accept piece of it

**Fix**:
- ✓ Keep scope focused
- ✓ Reference related proposals
- ✓ One major feature per SEP
- ✓ Create dependent SEPs if needed

### Mistake 5: Not Engaging with Community

**Problem**: Submit full proposal without discussing ideas first

**Impact**: Miss important feedback, end up redesigning

**Fix**:
- ✓ Discuss in GitHub issues first
- ✓ Get Core Maintainer input early
- ✓ Validate use cases with community
- ✓ Iterate on design before formal submission

### Mistake 6: Security Not Considered

**Problem**: Proposal introduces security issues

**Impact**: Required redesign or rejection

**Fix**:
- ✓ Include Security Implications section
- ✓ Think about attack surfaces
- ✓ Consider authentication/authorization
- ✓ Get security review from maintainers

### Mistake 7: Reference Implementation Too Minimal

**Problem**: Stub implementation that doesn't actually work

**Impact**: Can't reach Final status, questions about feasibility

**Fix**:
- ✓ Implement the entire feature
- ✓ Include error handling
- ✓ Add tests/examples
- ✓ Actually run the code

### Mistake 8: Unclear Technical Details

**Problem**: Spec uses vague language like "somehow" or "etc."

**Impact**: Confusing for implementors, hard to validate

**Fix**:
- ✓ Be precise about behavior
- ✓ Include actual message examples
- ✓ Document all error cases
- ✓ Use RFC 2119 keywords (MUST, SHOULD, etc.)

---

## Real-World Examples

### Example 1: SEP-1686 (Tasks)

**Status**: Final (Accepted)

**Type**: Standards Track

**Scope**: Enable long-running tool calls with status polling

**Timeline**: ~3 months from initial discussion to acceptance

**Key Success Factors**:
- ✓ Strong motivation with real use cases (6 detailed customer scenarios)
- ✓ Clear problem statement (no way to check tool call status)
- ✓ Comprehensive specification (new message types, state machine)
- ✓ Reference implementation in schema
- ✓ Community engagement (many comments, feedback incorporated)

**Lessons**:
- Real use cases from production systems convince maintainers
- Multiple customer scenarios show broad need
- Working implementation validates feasibility
- Detailed spec prevents implementation confusion

### Example 2: SEP-932 (Governance)

**Status**: Final (Accepted)

**Type**: Process

**Scope**: Formal governance structure, SEP process definition

**Timeline**: ~2 months

**Key Success Factors**:
- ✓ Clear problem (project grew too fast for informal governance)
- ✓ Detailed process with roles and responsibilities
- ✓ Inspiration from successful projects (Python, PyTorch, Rust)
- ✓ Core Maintainer sponsorship
- ✓ Implementation in documentation

**Lessons**:
- Process SEPs need clear procedures, not just motivation
- Reference successful projects for credibility
- Documentation is the "implementation" for process SEPs
- Maintainer support helps expedite acceptance

### Example 3: SEP-1699 (SSE Polling)

**Status**: Final (Accepted)

**Type**: Standards Track

**Scope**: Server-side disconnect support for SSE polling

**Key Success Factors**:
- ✓ Solves specific HTTP transport problem
- ✓ Backward compatible (optional feature)
- ✓ Simple, focused scope
- ✓ Reference implementation in transport layer

**Lessons**:
- Focused proposals easier to review and accept
- Backward compatibility increases acceptance likelihood
- Transport layer improvements less contentious than core protocol

---

## Technical Contribution Requirements

### Development Environment

**Required**:
```bash
Node.js 20+
npm 9+
Git
```

**Installation**:
```bash
# Use nvm for Node version management
nvm install  # uses .nvmrc to get correct version
npm install  # install dependencies
```

### Code Style & Quality

**TypeScript Schema**:
- Must follow existing conventions
- Type-safe (no `any`)
- Comprehensive JSDoc comments
- Prettier formatting: `npm run format:schema`

**Documentation (MDX)**:
- Prose wrap preserved
- Clear headings and structure
- Code examples functional
- Links verified

**All Code**:
- ESLint clean: `npm run check:schema`
- Prettier formatted: `npm run check:schema:ts`
- Type check passes: `npm run check:schema:ts`

### Running Validation

**Before submitting**:
```bash
# Full validation
npm run prep

# Individual checks
npm run check:schema:ts    # TypeScript validation
npm run check:schema:json  # JSON schema validation
npm run check:schema:examples  # Example validation
npm run check:docs         # Documentation validation
npm run generate:schema    # Generate from TypeScript
```

### Schema File Naming

**Location**: `schema/draft/schema.ts`

**Pattern** for new types:
```typescript
// Add to schema/draft/schema.ts
export interface NewFeature {
  /** Description of field */
  fieldName: string;
  /** Optional field */
  optionalField?: number;
}
```

**Documentation** generated automatically:
- JSON schema: `schema/draft/schema.json` (auto-generated)
- Markdown: `docs/specification/draft/schema.mdx` (auto-generated)

### Testing & Validation

**Automated Checks**:
- TypeScript compilation
- JSON schema generation
- Example validation
- Documentation links
- Code formatting

**Manual Testing**:
- Create integration test if applicable
- Document expected behavior
- Validate error cases
- Test with multiple implementations if possible

### Documentation Standards

**For Schema Changes**:
- JSDoc comments on interfaces
- Examples in specification docs
- Links to related features
- Error case documentation

**For New Primitives**:
- Explanation section
- Protocol flow diagram (if complex)
- Example messages
- Integration points

**For Process Changes**:
- Step-by-step procedures
- Role definitions
- Timeline/milestones
- Examples

---

## Checklist for SEP Submission

### Pre-Submission Checklist

- [ ] Idea validated with community
- [ ] SEP number requested and assigned
- [ ] Existing SEPs reviewed for conflicts
- [ ] Development environment set up
- [ ] Repository forked and cloned

### Content Checklist

**All SEPs**:
- [ ] Abstract (200 words)
- [ ] Motivation (real use cases)
- [ ] Rationale (design decisions)
- [ ] Clear, precise technical language
- [ ] Spell-checked and formatted

**Standards Track**:
- [ ] Detailed specification
- [ ] New message/type examples
- [ ] Backward compatibility section
- [ ] Security implications section
- [ ] Reference implementation planned

**Informational/Process**:
- [ ] Clear problem statement
- [ ] Practical guidance or procedures
- [ ] Examples or templates
- [ ] Community benefit explained

### Technical Checklist

- [ ] SEP file created: `seps/[NUMBER]-[slug].md`
- [ ] File formatted with correct header
- [ ] PR title: "SEP-XXX: [Title]"
- [ ] Branch name: `sep/[number]-[slug]`
- [ ] No conflicts with main branch
- [ ] Linked to GitHub issue (if applicable)

### Implementation Checklist (if applicable)

- [ ] Schema changes in `schema/draft/schema.ts`
- [ ] Generates without errors: `npm run generate:schema`
- [ ] Passes validation: `npm run check:schema`
- [ ] Documentation examples added
- [ ] Tests/examples work correctly
- [ ] Reference implementation linked

### Pre-Review Checklist

- [ ] Created branch from main
- [ ] Committed with clear commit messages
- [ ] Pushed to your fork
- [ ] Created PR with description
- [ ] Responded to any immediate feedback
- [ ] Available for questions/discussion

---

## Timeline Expectations

### Typical SEP Timeline

```
Week 1-2: Community Discussion
- Post idea in discussions/issues
- Get feedback on approach
- Request SEP number

Week 2-3: Draft & Initial Submission
- Write SEP using template
- Submit PR with draft
- Engage with early commenters

Week 3-5: Community Review
- Address feedback
- Iterate on specification
- Build reference implementation

Week 5-6: Reference Implementation
- Complete implementation
- Validate feasibility
- Add tests/examples

Week 6-7: Final Review
- Core Maintainers review
- Bi-weekly meeting discussion
- Address final concerns

Week 7-8: Acceptance & Merge
- Update status to "Final"
- Merge to main specification
- Announce to community
```

**Factors Affecting Timeline**:
- ✓ Proposal complexity (simple: 4 weeks, complex: 12+ weeks)
- ✓ Community alignment (high: shorter, low: longer)
- ✓ Implementation difficulty (easy: shorter, hard: longer)
- ✓ Maintainer availability (varies)
- ✓ Your responsiveness (faster response = faster review)

---

## Resources & References

### Official Documentation
- **SEP Guidelines**: https://modelcontextprotocol.io/community/sep-guidelines
- **Contributing**: https://github.com/modelcontextprotocol/specification/blob/main/CONTRIBUTING.md
- **Governance**: https://github.com/modelcontextprotocol/specification/blob/main/GOVERNANCE.md
- **Specification**: https://modelcontextprotocol.io/specification

### Communication Channels
- **GitHub Issues**: Feature requests and discussions
- **GitHub Discussions**: Community discussion
- **Discord**: (Check website for link)
- **Email**: maintainers@modelcontextprotocol.io (if needed)

### Related Reading
- **Python Enhancement Proposals (PEPs)**: https://www.python.org/dev/peps/
- **Rust RFCs**: https://github.com/rust-lang/rfcs
- **Language Server Protocol**: https://microsoft.github.io/language-server-protocol/

### Examples to Study
- SEP-932: Model Context Protocol Governance (Process)
- SEP-1686: Tasks (Standards Track, complex)
- SEP-1699: SSE Polling (Standards Track, focused)
- SEP-2133: Extensions (Standards Track, framework)

---

## Conclusion

The SEP process is designed to be **open, transparent, and community-driven**, while ensuring protocol quality through "rough consensus and running code."

**Key Success Factors**:
1. **Strong motivation** - Real problems with customer impact
2. **Clear specification** - Implementable without confusion
3. **Thoughtful design** - Rationale explaining decisions
4. **Working code** - Reference implementation validates feasibility
5. **Community engagement** - Responsive to feedback, collaborative

**Remember**:
- Start with discussion before full SEP
- Expect and welcome feedback
- Be willing to revise and improve
- Engage with maintainers early
- Think about long-term implications

**Final Advice**:
The MCP team is actively building this protocol and wants good contributions. They're accessible, responsive, and willing to work with you. Start the conversation early, share your ideas, and collaborate on making MCP better.

---

## Real-World Example: Content Negotiation SEP

This guide was used to develop **SEP-DRAFT-agent-content-negotiation.md**, a complete Standards Track proposal demonstrating the full SEP process:

**Process Stages Followed**:
1. ✅ **Preparation** - Researched RFC 2295, analyzed MCP architecture
2. ✅ **Draft & Discussion** - Created comprehensive specification
3. ✅ **Formal Submission** - Completed all required sections
4. ✅ **Community Review** - Specification ready for feedback (status: Draft)
5. ✅ **Reference Implementation** - TypeScript schemas and examples included
6. 🔄 **Final Review** - SEP request issued (#2291), awaiting maintainer review
7. ⏳ **Finalization** - Pending SEP number assignment

**SEP Quality Checklist Results**:
- ✅ All 7 required sections complete (Abstract through Reference Implementation)
- ✅ Feature tag vocabulary table (14 tags with detailed semantics)
- ✅ 3 initialize scenarios (agent, human, legacy)
- ✅ 8+ concrete tool response examples
- ✅ RFC 2295 mapping table (13 concepts analyzed)
- ✅ No breaking changes statement
- ✅ 5 security risks analyzed with mitigations
- ✅ TypeScript schema additions with JSDoc

**Key Takeaways from This SEP**:
1. **Strong motivation is essential** - Real-world use cases convinced spec quality
2. **RFC inspiration works** - Borrowing proven concepts from other domains strengthens proposals
3. **Reference implementation validates** - TypeScript examples made spec concrete and testable
4. **Backward compatibility matters** - Zero breaking changes increased proposal acceptance likelihood
5. **Security analysis is non-negotiable** - 5 risks analyzed prevented implementation issues

**Links**:
- **SEP Draft**: SEP-DRAFT-agent-content-negotiation.md (1,210 lines, production-ready)
- **SEP Request**: https://github.com/modelcontextprotocol/modelcontextprotocol/issues/2291
- **Repository**: https://github.com/schlpbch/agentic-content-negotiation
- **GitHub Release**: v0.9 (February 2026)

---

**Document Prepared**: February 22, 2026
**Last Updated**: February 22, 2026
**Accuracy**: Based on official MCP governance and SEP documentation
**Exemplified By**: SEP-DRAFT-agent-content-negotiation.md (Standards Track proposal)
