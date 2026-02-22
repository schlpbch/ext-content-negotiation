# RFC 2295: Transparent Content Negotiation in HTTP - Detailed Analysis

**Document Date**: February 22, 2026
**RFC**: 2295 (Experimental, March 1998)
**Authors**: Holtman and Mutz
**Topic**: Transparent Content Negotiation in HTTP

---

## Table of Contents

1. [Problem Statement & Motivation](#1-problem-statement--motivation)
2. [Technical Architecture](#2-technical-architecture)
3. [Variant Metadata Structure](#3-variant-metadata-structure)
4. [Negotiation Algorithms](#4-negotiation-algorithms)
5. [Feature Negotiation Framework](#5-feature-negotiation-framework)
6. [Implementation Considerations](#6-implementation-considerations)
7. [Caching & Entity Tags](#7-caching--entity-tags)
8. [Examples & Use Cases](#8-examples--use-cases)
9. [Design Limitations](#9-design-limitations)
10. [Relationship to HTTP Standards](#10-relationship-to-http-standards)

---

## 1. Problem Statement & Motivation

### Core Problem
HTTP/1.0 style content negotiation required sending exhaustive Accept headers with every request, creating significant bandwidth inefficiency across the internet. Since most resources have no variants, this approach was wasteful at scale.

### Key Benefits Sought
- **Graceful Evolution**: Support new web standards without forcing client upgrades
- **Platform Heterogeneity**: Accommodate diverse platforms from PDAs to VR systems
- **Eliminate User-Agent Sniffing**: Remove error-prone browser detection mechanisms
- **Internationalization**: Provide language selection without language bias
- **Cache Efficiency**: Move negotiation logic from exhaustive per-request headers to two-phase exchange

### Scalability Challenge
The specification identifies that "the biggest problem with this scheme is that it does not scale well." RFC 2295 addresses this by moving negotiation intelligence from exhaustive per-request headers to a structured, two-phase metadata exchange that proxies can participate in.

---

## 2. Technical Architecture

### Two-Phase Communication Model

RFC 2295 introduces a fundamentally different communication pattern compared to traditional HTTP content negotiation:

#### Phase 1: List Discovery
```
Client Request:
GET /paper HTTP/1.1
Negotiate: 1.0, guess-small

Server Response:
HTTP/1.1 300 Multiple Choices
Alternates: {"paper.en" 0.9 {type text/html} {language en}},
            {"paper.fr" 0.7 {type text/html} {language fr}},
            {"paper.de" 0.8 {type text/html} {language de}}
```

- Client requests a negotiable resource
- Server responds with 300 Multiple Choices status
- Alternates header lists all available variants with attributes
- Response includes machine-readable metadata + human-readable HTML fallback

#### Phase 2: Variant Retrieval
```
Client Request:
GET /paper.en HTTP/1.1

Server Response:
HTTP/1.1 200 OK
Content-Type: text/html
Content-Language: en
[entity body]
```

- User agent's local algorithm selects optimal variant from list
- Client retrieves selected variant via standard GET request
- Second response is normal HTTP—transparent to intermediate proxies

#### Optimization Path: Choice Responses
If initial request includes small Accept headers or specific Negotiate directives:
```
Client Request:
GET /paper HTTP/1.1
Accept: text/html, application/pdf
Accept-Language: en;q=1.0, fr;q=0.8
Negotiate: 1.0

Server Response (Choice):
HTTP/1.1 200 OK
Content-Type: text/html
Content-Language: en
Content-Location: /paper.en
TCN: choice
[Alternates header optional]
[entity body]
```

Server executes remote variant selection algorithm and returns chosen variant directly in single transaction.

### Three Response Types

1. **List Responses**: Return variant metadata without entity body (always permitted)
2. **Choice Responses**: Return selected variant plus optional metadata (conditional on Negotiate header)
3. **Adhoc Responses**: Custom responses for compatibility workarounds (exceptional)

---

## 3. Variant Metadata Structure

### Variant Description Format

```
{"paper.1" 0.9 {type text/html} {language en}}
```

Breaking down the structure:
- **URI**: `paper.1` - resource identifier for this variant
- **Source-Quality**: `0.9` - quality metric (0.0-1.0)
- **Attributes**: `{type ...} {language ...}` - variant properties

### Source-Quality Definition

**Critical Distinction**: Source-quality expresses "quality of the variant, as a representation of the negotiable resource, when this variant is rendered with a perfect rendering engine."

**Key Point**: Servers should NOT account for transmission delays or variant size. The specification explicitly states: "Servers should not account for the size of the variant and its impact on transmission and rendering delays; the size of the variant should be stated in the length attribute."

This separation ensures quality metrics reflect content fidelity, not network performance.

### Standard Attributes

#### Type
- MIME media type (application/pdf, text/html, etc.)
- Separated from charset for precision
- Example: `{type text/html}`

#### Charset
- Character set encoding
- Separated from type attribute
- Example: `{charset UTF-8}`

#### Language
- RFC 1766 language tags
- Supports multiple values
- Examples: `{language en}`, `{language en;q=0.8 fr;q=0.5}`

#### Length
- Variant-only size, excluding embedded objects
- Useful for user agents to estimate transmission time
- Example: `{length 5236}`

#### Description
- User-facing text describing the variant
- Optional language specification
- Example: `{description "French translation" {language en}}`

### Extension Mechanism

Attributes beginning with `x-` enable experimentation without namespace collisions:
```
{x-feature-flags rendering-v2}
{x-optimization performance-tier-1}
```

This allows server implementers to negotiate custom properties while standards develop separately.

---

## 4. Negotiation Algorithms

### Local Algorithm (User Agent)

The user agent maintains a feature set recording capabilities and preferences. Upon receiving a variant list, it performs these steps:

1. **Parse Alternates Header**: Extract all variant descriptions
2. **Compute Quality Scores**: For each variant, multiply component quality factors:
   - Type match quality
   - Language preference quality
   - Character set preference quality
   - Feature compatibility quality
   - Size penalty factor (optional)
3. **Select Best Variant**: Choose variant with highest overall quality
4. **Retrieve Variant**: Issue GET request to selected variant URI

**Example Calculation**:
```
Variant: {type text/html} {language en}
Quality = (type_match_q=1.0) × (language_q=1.0) = 1.0

Variant: {type text/plain} {language en}
Quality = (type_match_q=0.1) × (language_q=1.0) = 0.1

Selection: First variant chosen
```

### Remote Algorithm (Server)

Standardized algorithms execute at origin server or proxy level when permitted by Negotiate headers.

**Algorithm Invocation**:
```
Negotiate: 1.0, guess-small
```

- `1.0` = algorithm version 1.0
- `guess-small` = implementation hint (when uncertain, guess toward smaller transmission)

**Algorithm Responsibility**:
"A remote algorithm typically computes whether the Accept- headers in the request contain sufficient information to allow a choice, and if so, which variant is the best variant."

### Backward Compatibility Guarantee

Remote algorithms versioned as X.Y must maintain strict backward compatibility:

"A remote algorithm from version X.Y MUST be downwards compatible with all algorithms from X.0 up to X.Y. Downwards compatibility means that, if supplied with the same information, the newer algorithm MUST make the same choice, or a better choice, as the old algorithm."

This ensures that clients can safely enable newer algorithms while maintaining deterministic behavior.

### Security Constraint: Neighboring Variants

Servers may only return choice responses for "neighboring variants"—those whose URI shares the same directory path:

```
Negotiable Resource: /path/to/paper
Valid Neighboring Variants: /path/to/paper.en, /path/to/paper.fr
Invalid Neighbors: /path/other/paper.en, /paper.en
```

**Rationale**: Prevents spoofing attacks where an attacker's variant resources could become transparently negotiable. This boundary trades negotiation flexibility for security.

---

## 5. Feature Negotiation Framework

### Motivation

The four standard dimensions (type, charset, language, length) don't cover all negotiable properties. Features extend negotiation to include:
- Display capabilities (tables, frames, JavaScript)
- Color depth and screen resolution
- Audio/video codecs
- Custom application-specific properties

### Feature Tags

**Definition**: Identifiers for negotiable properties

**Examples**:
- `tables` - supports HTML tables
- `frames` - supports HTML frames
- `colordepth` - color depth capability
- `screenwidth` - display width
- `javascript` - JavaScript support

**Case-Sensitivity**: "Feature tag comparison is case-insensitive. A token tag XYZ is equal to a quoted-string tag "XYZ"."

### Feature Values and Predicates

#### Simple Presence
```
{features tables frames}
```
True if client supports both tables AND frames.

#### Negation
```
{features !frames !javascript}
```
True if client does NOT support frames AND does NOT support JavaScript.

#### Equality Comparison
```
{features colordepth=24}
{features paper=A4}
```
True if colordepth equals 24 OR paper size equals A4.

#### Range Matching
```
{features colordepth=[24-]}
{features screenwidth=[1024-1920]}
```
True if colordepth >= 24 OR screenwidth between 1024-1920.

### Feature Lists with Quality Degradation

Variants can express quality adjustments based on feature availability:

```
{features !textonly [blebber !wolx] colordepth=3;+0.7}
```

Breaking down:
- `!textonly` - Basic requirement (NOT text-only mode)
- `[blebber !wolx]` - Feature group requiring `blebber` support and NOT `wolx`
- `colordepth=3;+0.7` - Apply +0.7 quality boost if colordepth=3

**Aggregation Rule**: "A user agent SHOULD, and a remote variant selection algorithm MUST compute the quality degradation factor associated with the features attribute by multiplying all quality degradation factors of the elements of the feature-list."

### Accept-Features Header

Clients communicate their feature capabilities:

```
Accept-Features: tables, frames, !javascript, colordepth=24
Accept-Features: colordepth=[16-256], screenwidth=[640-]
Accept-Features: *
```

**Wildcard Semantics**: The `*` wildcard indicates incomplete capability disclosure. "If the user agent sends Accept-Features: *, then it does not communicate which optional features are supported."

**Precise Values**: `colordepth={5}` indicates exact value precision (colordepth is exactly 5).

---

## 6. Implementation Considerations

### Server-Side Implementation

Servers face three implementation paths:

#### Path 1: CGI Script Wrapper
```
negotiable_resource.cgi:
1. Parse Negotiate header
2. Determine if remote selection is permitted
3. Consult variant metadata
4. Execute algorithm
5. Return list or choice response
```

Advantages: Works with any HTTP server
Disadvantages: Performance overhead per request

#### Path 2: Native Server Modules
HTTP servers (Apache, IIS, etc.) implement negotiation logic as core modules.

Advantages: Performance optimization, tight integration
Disadvantages: Server-specific implementation required

#### Path 3: Publishing Tool Integration
Content management systems generate appropriate headers during deployment.

Advantages: Centralized management, consistency
Disadvantages: Requires CMS support

### Server Requirements

Origin servers must:
1. "Be able to send a list response when getting a GET request on the resource"
2. Ensure variant resources "MUST NOT engage in transparent content negotiation itself" (preventing recursive negotiation)
3. Maintain consistency between variant descriptions and actual variant resources

### User Agent Implementation

Clients must:
1. **Implement local selection logic** - Parse Alternates headers, compute quality scores
2. **Send Negotiate headers** - Indicate algorithm version support and hints
3. **Handle all response types** - Support list, choice, and adhoc responses
4. **Verify alignment** - "If a remote variant selection algorithm which was enabled could have made a choice different from the choice the local algorithm would make, the user agent MAY apply its local algorithm to any variant list in the response, and automatically retrieve and display another variant if the local algorithm makes a different choice."

### Proxy Implementation

Transparent HTTP/1.1 proxies automatically support basic negotiation:
1. Extract and cache Alternates headers
2. Recognize negotiable resources by 300 response status
3. For cached variant lists, retrieve appropriate variant on behalf of client
4. Advanced proxies can enable remote algorithms and construct choice responses

---

## 7. Caching & Entity Tags

### Problem: Cache Invalidation for Variants

Traditional entity tags don't adequately address the case where:
- Entity body changes → old variant becomes stale
- Variant list changes → different variants become available
- Either change requires different cache strategies

### Structured Entity Tags Solution

RFC 2295 extends entity tags with variant list validators:

**Format**:
```
"etag;vlv"                    (Strong tag with validator)
W/"etag;vlv"                  (Weak tag with validator)
```

**Definition**: "A structured entity tag consists of a normal entity tag of which the opaque string is extended with a semicolon followed by the text (without the surrounding quotes) of a variant list validator."

### Validation Semantics

**Normal Tag** (`etag`):
- Validates entity body
- Validates non-Alternates entity headers

**Variant List Validator** (`vlv`):
- Validates Alternates header contents
- Enables proxies to detect variant list staleness independently

**Independence Example**:
```
Response A: ETag: "abc;vlv1"
Response B: ETag: "def;vlv1"

Interpretation: Same variant list, different body content
Proxy Action: Body has changed but variants remain valid
```

### Variant List Validator Semantics

"If two responses contain the same variant list validator, a cache can treat the Alternates headers in these responses as equivalent (though the headers themselves need not be identical)."

This enables:
- Efficient cache reuse
- Detection of minor Alternates changes
- Independent freshness tracking for metadata vs. entity

### Freshness Boundaries

Cached variant lists have a maximal lifetime M:

**Calculation**: "M = maximum of all freshness lifetimes assigned (using max-age directives or Expires headers) by the origin server to:
a) The responses from the negotiable resource itself
b) The responses from its neighboring variant resources"

**Example**:
```
Negotiable resource Cache-Control: max-age=3600
Variant .en Cache-Control: max-age=86400
Variant .fr Cache-Control: max-age=7200

M = 86400 seconds (largest value)
Variant list valid for at most 86400 seconds
```

This prevents variant list staleness while respecting per-resource freshness policies.

---

## 8. Examples & Use Cases

### Use Case 1: Multi-Language Resource

**Server Configuration**:
```
Resource: /documents/manual
Variants:
  - /documents/manual.en (English)
  - /documents/manual.fr (French)
  - /documents/manual.de (German)
```

**Request/Response**:
```
Client Request:
GET /documents/manual HTTP/1.1
Accept-Language: en;q=1.0, fr;q=0.8, de;q=0.5
Negotiate: 1.0

Server Response:
HTTP/1.1 300 Multiple Choices
Alternates: {"manual.en" 0.9 {type text/html} {language en}},
            {"manual.fr" 0.7 {type text/html} {language fr}},
            {"manual.de" 0.8 {type text/html} {language de}}
```

**Client Decision**: Multiplies type match (1.0) × language preference (1.0 for en) = 1.0
**Result**: Client retrieves manual.en

---

### Use Case 2: Format Degradation

**Scenario**: Original content in PDF, converted to lower-fidelity formats

```
Server Response:
HTTP/1.1 300 Multiple Choices
Alternates: {"report.pdf" 1.0 {type application/pdf}},
            {"report.html" 0.8 {type text/html}},
            {"report.txt" 0.3 {type text/plain}}
```

**Source-Quality Values Interpretation**:
- PDF: 1.0 (original, perfect fidelity)
- HTML: 0.8 (good fidelity, but loses some formatting)
- Text: 0.3 (significant loss of tables, formatting)

**Client Logic**:
```
If (Accept: application/pdf):
  Quality = 1.0 × 1.0 = 1.0 → Select PDF

If (Accept: text/*):
  HTML Quality = 0.8 × 1.0 = 0.8
  Text Quality = 0.3 × 1.0 = 0.3
  → Select HTML
```

---

### Use Case 3: Feature-Based Selection

**Scenario**: Rich UI for capable browsers, fallback for minimal browsers

```
Server Response:
HTTP/1.1 300 Multiple Choices
Alternates: {"index-full.html" 1.0 {type text/html} {features tables frames}},
            {"index-basic.html" 0.7 {type text/html} {features !tables !javascript}}
```

**Client with Full Features**:
```
Accept-Features: tables, frames, javascript, css3
Computation: index-full.html matches all features → 1.0
Selection: index-full.html
```

**Client with Minimal Features**:
```
Accept-Features: !tables, !javascript
Computation: index-basic.html matches constraints → 0.7
Selection: index-basic.html
```

---

### Use Case 4: Choice Response Construction

**Scenario**: Server makes selection on behalf of client

```
Client Request:
GET /report HTTP/1.1
Accept: application/pdf, text/html;q=0.9
Accept-Language: en
Negotiate: 1.0, guess-small

Server Response (Choice):
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Language: en
Content-Location: /report.pdf
TCN: choice
Vary: Accept, Accept-Language
[PDF entity body]
```

**Key Headers**:
- `TCN: choice` - Indicates transparent content negotiation applied
- `Content-Location: /report.pdf` - Specifies which variant was selected
- `Vary: Accept, Accept-Language` - Informs caches of negotiation dimensions

---

## 9. Design Limitations

### Variant List Length

"A typical transparently negotiable resource will have 2 to 10 variants, depending on its purpose."

**Rationale**:
- Cache efficiency decreases with list size
- User interface usability for manual fallback selection
- Practical proxy cache performance
- Administrative burden increases

**Impact**: Design discourages highly granular negotiation (e.g., 100+ variants per resource).

---

### Remote Algorithm Constraints

Servers may only return choice responses for "neighboring variants" (same directory):

```
Acceptable: /path/to/resource.en (neighbor of /path/to/resource)
Unacceptable: /other/resource.en (different directory)
```

**Trade-off**: Sacrifices negotiation flexibility for security (prevents variant spoofing).

---

### Content Encoding Exclusion

"Negotiation on the content encoding of a response (gzipped, compressed, etc.) is left outside of the realm of transparent negotiation."

**Rationale**: Content encoding is:
- Transport-layer concern (independent of content semantics)
- Handled separately by Accept-Encoding header
- Less beneficial for quality-based selection

**Impact**: Transparent negotiation focuses on semantic variants, not transmission encodings.

---

### HTTP/1.0 Compatibility

The specification accommodates legacy HTTP/1.0 proxies:

```
Cache-Control: no-cache, max-age=0
Expires: <past date>
```

**Strategy**: Dual-header approach
- HTTP/1.0 proxies respect Expires
- HTTP/1.1 proxies prefer Cache-Control

**Trade-off**: Slightly more verbose cache directives for broader compatibility.

---

### Feature Tag Standardization Dependency

Rather than defining specific feature tags in the RFC, the specification creates a framework for registration:

```
Defined: tables, frames, colordepth, etc. (external registry)
Not Defined: Application-specific features
```

**Trade-off**:
- **Advantage**: Enables extensibility without standards bottlenecks
- **Disadvantage**: Requires external coordination for tag definitions
- **Risk**: Fragmented implementations without centralized guidance

---

## 10. Relationship to HTTP Standards

### HTTP/1.1 Foundation

RFC 2295 layers atop HTTP/1.1, reusing and extending core mechanisms:

**Reused Mechanisms**:
- Entity tags (extended with validators)
- Vary headers (elaborated for negotiation contexts)
- Cache-Control directives
- Conditional request headers (If-None-Match)
- Content-Location header

**Extended Mechanisms**:
- Entity tags: Normal tag + variant list validator
- Negotiate header: Explicit negotiation protocol version and hints
- Alternates header: Machine-readable variant descriptions
- TCN header: Indicates choice response status

### Backward Compatibility

"Use of this extension does not require use of HTTP/1.1: transparent content negotiation can also be done if some or all of the parties are HTTP/1.0 systems."

**Compatibility Mechanisms**:
- Alternates header (HTTP/1.0 compatible in format)
- Fallback to 300 response (HTTP/1.0 understands)
- Dual Cache-Control/Expires headers
- Choice responses optional (list responses always available)

### Coexistence with Other Negotiation

Transparent negotiation coexists with, rather than replaces:

**Client-Driven Negotiation**:
```
Accept: text/html, application/xhtml+xml
Accept-Language: en-US,en;q=0.9
```
Traditional HTTP/1.1 approach; orthogonal to transparent negotiation.

**Server-Driven Selection**:
```
Vary: Accept
```
Server guesses based on client headers without explicit exchange.

**Proprietary Negotiation**:
```
Set-Cookie: preferred_format=html
```
Application-level content selection mechanisms.

**Layered Approach**: The specification states: "It is intended that transparent negotiation can co-exist with other negotiation schemes, both open and proprietary, which cover different application domains."

### Feature Tag Registry

The specification anticipates "an effort to define a protocol-independent registry for feature tags," positioning itself as one client of a broader standardization ecosystem rather than a standalone protocol.

**Registry Role**:
- Maintain canonical feature tag definitions
- Prevent namespace collisions
- Document feature semantics
- Enable cross-application feature compatibility

---

## Conclusion

RFC 2295 represents an early (experimental) attempt to solve scalable content negotiation through structural metadata exchange and optional remote selection.

**Core Contribution**: By separating negotiation metadata from request/response cycles and enabling proxy participation, it addresses HTTP/1.0's per-request overhead problem.

**Key Design Principles**:
- Two-phase exchange minimizes negotiation overhead
- Local algorithms preserve client autonomy
- Structured metadata enables proxy optimization
- Security boundaries prevent variant spoofing
- Backward compatibility accommodates legacy infrastructure

**Pragmatic Compromises**:
- Conservative variant list sizing (2-10 variants typical)
- Reliance on external feature tag standardization
- Exclusion of content encoding from negotiation scope
- Neighboring-variant restriction for remote selection

**Historical Impact**: While not universally adopted in production (partly due to complexity), RFC 2295 established foundational concepts for content negotiation that influenced subsequent work in HTTP content negotiation, REST API versioning, and adaptive content delivery systems.

**Modern Relevance**: With the rise of adaptive media delivery, mobile-first design, and API versioning, transparent negotiation's core insights—metadata-driven selection, proxy participation, and structured capability exchange—remain relevant to contemporary problems in web architecture.

---

## Application to MCP

This analysis directly informed the development of **SEP-DRAFT-agent-content-negotiation.md**, which proposes transparent content negotiation for the Model Context Protocol.

**Key Concepts Adopted**:
- Feature tag syntax (presence, negation, equality predicates)
- Session-scoped capability declaration
- Server-side variant selection based on client capabilities
- Graceful fallback for unsupported features

**Key Concepts Adapted**:
- Simplified from RFC 2295's two-phase exchange to single-phase (MCP's stateful model)
- Feature tags mirror MCP capabilities (sampling, elicitation, roots, tasks)
- Removed complex caching and entity tag validation (not applicable to MCP)

**Links**:
- **SEP Draft**: SEP-DRAFT-agent-content-negotiation.md
- **SEP Request**: https://github.com/modelcontextprotocol/modelcontextprotocol/issues/2291
- **Repository**: https://github.com/schlpbch/agentic-content-negotiation

---

**Analysis Prepared**: February 22, 2026
**Source**: RFC 2295, IETF
**Status**: Experimental (1998)
**Application Status**: Concepts applied to MCP SEP draft (February 2026)
