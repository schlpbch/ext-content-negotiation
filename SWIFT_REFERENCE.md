# Swift Reference Implementation

This document provides a Swift reference implementation of the
`io.modelcontextprotocol/content-negotiation` extension using the
[official MCP Swift SDK](https://github.com/modelcontextprotocol/swift-sdk).

## Overview

The implementation has three layers:

1. **`Features`** — a Swift `struct` with convenience properties for the most
   common negotiation axes, mirroring the Python `Features` Pydantic model and
   TypeScript `Features` class.
2. **`parseFeatures(from:)`** — parses feature tags from `Client.Capabilities`
   using pattern matching on the SDK's `Value` enum.
3. **`ContentNegotiationState`** — a Swift `actor` that captures the negotiated
   `Features` during the `initializeHook` callback and makes them available
   inside tool handlers via closure capture.

> **Why the actor?** Tool handler closures passed to `withMethodHandler` do not
> receive client capabilities directly — `clientCapabilities` is actor-isolated
> on `Server`. The `initializeHook` provides the only direct access point; an
> `actor` stores the result safely for concurrent handler use.

## Dependencies

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/modelcontextprotocol/swift-sdk.git", from: "0.11.0")
],
targets: [
    .executableTarget(
        name: "WeatherServer",
        dependencies: [.product(name: "MCP", package: "swift-sdk")]
    )
]
```

## Implementation

```swift
// ContentNegotiation.swift
import MCP

private let extensionID = "io.modelcontextprotocol/content-negotiation"

/// Negotiated content preferences for the current MCP session.
/// All properties return safe defaults when no negotiation took place.
struct Features {

    let tags: [String]

    /// Sentinel returned when no negotiation took place.
    static let empty = Features(tags: [])

    /// True when the client declared the `agent` tag.
    var isAgent: Bool { tags.contains("agent") }

    /// True when the client declared the `human` tag.
    var isHuman: Bool { tags.contains("human") }

    /// True when `interactive` is present and `!interactive` is absent.
    var isInteractive: Bool {
        tags.contains("interactive") && !tags.contains("!interactive")
    }

    /// True when the client declared the `sampling` tag.
    var hasSampling: Bool { tags.contains("sampling") }

    /// True when the client declared the `elicitation` tag.
    var hasElicitation: Bool { tags.contains("elicitation") }

    /// Returns the negotiated format: `"json"`, `"text"`, or `"markdown"` (default).
    var format: String {
        tags.lazy
            .compactMap { tag -> String? in
                guard tag.hasPrefix("format=") else { return nil }
                let v = String(tag.dropFirst("format=".count))
                return ["json", "text", "markdown"].contains(v) ? v : nil
            }
            .first ?? "markdown"
    }

    /// Returns the negotiated verbosity: `"compact"`, `"standard"` (default), or `"verbose"`.
    var verbosity: String {
        tags.lazy
            .compactMap { tag -> String? in
                guard tag.hasPrefix("verbosity=") else { return nil }
                let v = String(tag.dropFirst("verbosity=".count))
                return ["compact", "standard", "verbose"].contains(v) ? v : nil
            }
            .first ?? "standard"
    }

    /// Check for any feature tag, including vendor tags (`x-*`).
    func hasTag(_ tag: String) -> Bool { tags.contains(tag) }

    /// Return the value of a `prefix=value` vendor tag, or `nil`.
    ///
    /// Example: `features.vendorValue(for: "x-mycompany-hint")` returns `"value"`
    /// when the tag `x-mycompany-hint=value` is present.
    func vendorValue(for prefix: String) -> String? {
        let key = prefix + "="
        guard let tag = tags.first(where: { $0.hasPrefix(key) }) else { return nil }
        return String(tag.dropFirst(key.count))
    }
}

/// Parse feature tags from `Client.Capabilities`.
///
/// Falls back to `Features.empty` (all defaults) when no negotiation took
/// place or the extension was not declared by the client.
///
/// **Note on `experimental` vs `extensions`**: The MCP Swift SDK maps
/// `Client.Capabilities` from JSON. The `extensions` key in the JSON wire
/// format lands in the `experimental: [String: Value]?` property — both are
/// open-ended property maps in the schema. This function reads the extension
/// from `experimental` accordingly.
func parseFeatures(from capabilities: Client.Capabilities?) -> Features {
    guard
        let experimental = capabilities?.experimental,
        let extValue = experimental[extensionID],
        case .object(let extMap) = extValue,
        let featuresValue = extMap["features"],
        case .array(let featuresArr) = featuresValue
    else { return .empty }

    let tags = featuresArr.compactMap { value -> String? in
        guard case .string(let s) = value else { return nil }
        return s
    }
    return Features(tags: tags)
}

/// Actor that holds the negotiated `Features` for the current session.
///
/// Populated in `initializeHook` during the `initialize` handshake;
/// consumed inside tool handlers via closure capture. Thread-safe by
/// Swift's actor isolation model.
actor ContentNegotiationState {
    private(set) var features: Features = .empty

    func set(_ features: Features) {
        self.features = features
    }
}
```

## Server Example

```swift
// WeatherServer.swift
import Foundation
import MCP

@main
struct WeatherServer {
    static func main() async throws {
        let server = Server(
            name: "Weather Service",
            version: "1.0.0",
            capabilities: Server.Capabilities(tools: .init())
        )

        // Capture negotiated features from the initialize handshake.
        let negotiationState = ContentNegotiationState()

        // Register the tool list.
        await server.withMethodHandler(ListTools.self) { _ in
            ListTools.Result(tools: [
                Tool(
                    name: "get_weather",
                    description: "Return weather data for a location.",
                    inputSchema: .object([
                        "type":       .string("object"),
                        "properties": .object([
                            "location": .object([
                                "type":        .string("string"),
                                "description": .string("City or region name"),
                            ])
                        ]),
                        "required": .array([.string("location")]),
                    ])
                )
            ])
        }

        // Register the tool call handler.
        await server.withMethodHandler(CallTool.self) { [negotiationState] params in
            guard params.name == "get_weather" else {
                return CallTool.Result(content: [], isError: true)
            }

            let features = await negotiationState.features

            let location: String
            if case .string(let s) = params.arguments?["location"] {
                location = s
            } else {
                location = "unknown"
            }

            // Raw data — always computed the same way regardless of negotiation
            let data: [String: Any] = [
                "location":                  location,
                "temperature_c":             8,
                "humidity_percent":          72,
                "precipitation_probability": 0.30,
                "wind_speed_kmh":            15,
                "uv_index":                  2,
            ]

            let text: String
            if features.isAgent && features.format == "json" {
                if features.verbosity == "compact" {
                    // Compact structured payload for agents
                    text = toJSON(data)
                } else {
                    // Verbose: add units and metadata
                    var verbose = data
                    verbose["units"]             = "metric"
                    verbose["source"]            = "WeatherAPI"
                    verbose["valid_for_minutes"] = 60
                    text = toJSON(verbose)
                }
            } else {
                // Human-readable markdown
                var lines = [
                    "## Weather in \(location)",
                    "- **Temperature**: \(data["temperature_c"]!)°C",
                    "- **Humidity**: \(data["humidity_percent"]!)%",
                    "- **Precipitation**: \(Int(0.30 * 100))% chance",
                    "- **Wind**: \(data["wind_speed_kmh"]!) km/h",
                ]
                if features.verbosity == "verbose" {
                    lines += [
                        "- **UV Index**: \(data["uv_index"]!) (low)",
                        "",
                        "_Data provided by WeatherAPI. Valid for 60 minutes._",
                    ]
                }
                text = lines.joined(separator: "\n")
            }

            return CallTool.Result(content: [.init(type: .text, text: text)])
        }

        // Start server; capture capabilities in the initialize hook.
        let transport = StdioTransport()
        try await server.start(
            transport: transport,
            initializeHook: { [negotiationState] _, capabilities in
                let features = parseFeatures(from: capabilities)
                await negotiationState.set(features)
            }
        )
    }
}

// MARK: - Helpers

private func toJSON(_ dict: [String: Any]) -> String {
    guard
        let data = try? JSONSerialization.data(
            withJSONObject: dict, options: [.sortedKeys]),
        let string = String(data: data, encoding: .utf8)
    else { return "{}" }
    return string
}
```

## Client Initialization Example

```swift
// ClientExample.swift — declares content negotiation at initialize
import MCP

let client = Client(
    name: "my-agent",
    version: "1.0.0",
    capabilities: Client.Capabilities(
        experimental: [
            "io.modelcontextprotocol/content-negotiation": .object([
                "version":  .string("1.0"),
                "features": .array([
                    .string("agent"),
                    .string("sampling"),
                    .string("format=json"),
                    .string("verbosity=compact"),
                ]),
            ])
        ]
    )
)

let transport = try StdioTransport(
    command: "/path/to/weather-server",
    arguments: []
)
try await client.connect(transport: transport)

let result = try await client.callTool(
    named: "get_weather",
    arguments: ["location": .string("Bern")]
)
print(result.content.first?.text ?? "") // → compact JSON string
```

## Design Notes

- **`initializeHook` pattern**: Tool handlers registered with
  `withMethodHandler` are `@Sendable` closures that do not receive client
  capabilities directly. The `initializeHook` callback on `server.start` is the
  only guaranteed access point for `Client.Capabilities`. Storing the result in
  a `ContentNegotiationState` actor and capturing it in the tool handler closure
  is the idiomatic Swift 6 pattern.
- **Swift `actor` for safe concurrency**: `ContentNegotiationState` is an `actor`,
  giving Swift's concurrency model automatic data-race protection. All reads and
  writes are actor-isolated and require `await`.
- **`Value` enum pattern matching**: The SDK represents JSON as a custom `Value`
  enum (`.null`, `.bool`, `.int`, `.double`, `.string`, `.array`, `.object`).
  `parseFeatures` uses `guard case` pattern matching at each level; any
  unexpected shape falls through to `Features.empty`.
- **`experimental` vs `extensions`**: The MCP Swift SDK maps
  `Client.Capabilities` from JSON. The `extensions` key in the JSON wire format
  lands in the `experimental: [String: Value]?` property — both are open-ended
  property maps in the schema. `parseFeatures` reads from `experimental` and
  documents this mapping explicitly.
- **Server capability advertisement**: The current `Server.Capabilities` struct
  does not expose an `experimental` or `extensions` field. Advertising extension
  support to clients is not directly expressible in the Swift SDK at this time;
  track [swift-sdk](https://github.com/modelcontextprotocol/swift-sdk) for
  updates.
- **Safe defaults**: all `Features` properties return sensible defaults
  (`format` → `"markdown"`, `verbosity` → `"standard"`) so tools degrade
  gracefully when no negotiation occurred.
- **Security**: feature tags only influence content shape — the tool always
  computes the same underlying data and only varies the presentation. Tags never
  control authentication or access decisions.
- **Extensible**: `Features.hasTag(_:)` and `Features.vendorValue(for:)` support
  vendor tags (`x-mycompany-hint=value`) without any changes to `parseFeatures`.
