# Kotlin Reference Implementation

This document provides a Kotlin reference implementation of the
`io.modelcontextprotocol/content-negotiation` extension using the
[official MCP Kotlin SDK](https://github.com/modelcontextprotocol/kotlin-sdk).

## Overview

The implementation has two layers:

1. **`Features`** — a Kotlin `data class` with convenience properties for the
   most common negotiation axes, mirroring the Python `Features` Pydantic model
   and TypeScript `Features` class.
2. **`getFeatures(ClientCapabilities?)`** — parses feature tags from the client
   capabilities available on the `ClientConnection` receiver inside every
   `addTool` handler.

## Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.modelcontextprotocol:kotlin-sdk:0.8.3")
}
```

## Implementation

```kotlin
// ContentNegotiation.kt
package com.example.mcp

import io.modelcontextprotocol.kotlin.sdk.ClientCapabilities
import kotlinx.serialization.json.JsonArray
import kotlinx.serialization.json.JsonObject
import kotlinx.serialization.json.JsonPrimitive
import kotlinx.serialization.json.contentOrNull

const val EXTENSION_ID = "io.modelcontextprotocol/content-negotiation"

/**
 * Negotiated content preferences for the current MCP session.
 * All properties return safe defaults when no negotiation took place.
 */
data class Features(val tags: List<String> = emptyList()) {

    companion object {
        /** Sentinel returned when no negotiation took place. */
        val EMPTY = Features()
    }

    /** True when the client declared the `agent` tag. */
    val isAgent: Boolean get() = "agent" in tags

    /** True when the client declared the `human` tag. */
    val isHuman: Boolean get() = "human" in tags

    /** True when `interactive` is present and `!interactive` is absent. */
    val isInteractive: Boolean get() = "interactive" in tags && "!interactive" !in tags

    /** True when the client declared the `sampling` tag. */
    val hasSampling: Boolean get() = "sampling" in tags

    /** True when the client declared the `elicitation` tag. */
    val hasElicitation: Boolean get() = "elicitation" in tags

    /**
     * Returns the negotiated format: `"json"`, `"text"`, or `"markdown"` (default).
     */
    val format: String get() {
        for (tag in tags) {
            if (tag.startsWith("format=")) {
                val v = tag.removePrefix("format=")
                if (v in setOf("json", "text", "markdown")) return v
            }
        }
        return "markdown"
    }

    /**
     * Returns the negotiated verbosity: `"compact"`, `"standard"` (default),
     * or `"verbose"`.
     */
    val verbosity: String get() {
        for (tag in tags) {
            if (tag.startsWith("verbosity=")) {
                val v = tag.removePrefix("verbosity=")
                if (v in setOf("compact", "standard", "verbose")) return v
            }
        }
        return "standard"
    }

    /** Check for any feature tag, including vendor tags (`x-*`). */
    fun hasTag(tag: String): Boolean = tag in tags

    /**
     * Return the value of a `prefix=value` vendor tag, or `null`.
     *
     * Example: `features.vendorValue("x-mycompany-hint")` returns `"value"`
     * when the tag `x-mycompany-hint=value` is present.
     */
    fun vendorValue(prefix: String): String? {
        val key = "$prefix="
        return tags.firstOrNull { it.startsWith(key) }?.removePrefix(key)
    }
}

/**
 * Return the negotiated [Features] for the current session.
 *
 * Falls back to [Features.EMPTY] (all defaults) when no negotiation took place
 * or the extension was not declared by the client.
 *
 * **Note on `experimental` vs `extensions`**: The MCP Kotlin SDK maps
 * `ClientCapabilities` from JSON. The `extensions` key in the JSON wire format
 * lands in the `experimental` property of the Kotlin data class — both are
 * open-ended property maps in the schema. This function reads the extension
 * from `experimental` accordingly.
 */
fun getFeatures(capabilities: ClientCapabilities?): Features {
    return try {
        val experimental = capabilities?.experimental ?: return Features.EMPTY
        val ext = experimental[EXTENSION_ID] as? JsonObject ?: return Features.EMPTY
        val featuresArr = ext["features"] as? JsonArray ?: return Features.EMPTY
        val tags = featuresArr
            .filterIsInstance<JsonPrimitive>()
            .mapNotNull { it.contentOrNull }
        Features(tags)
    } catch (e: Exception) {
        Features.EMPTY
    }
}
```

## Server Example

```kotlin
// WeatherServer.kt
package com.example.mcp

import io.modelcontextprotocol.kotlin.sdk.Implementation
import io.modelcontextprotocol.kotlin.sdk.ServerCapabilities
import io.modelcontextprotocol.kotlin.sdk.TextContent
import io.modelcontextprotocol.kotlin.sdk.server.Server
import io.modelcontextprotocol.kotlin.sdk.server.ServerOptions
import io.modelcontextprotocol.kotlin.sdk.server.StdioServerTransport
import kotlinx.coroutines.runBlocking
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json
import kotlinx.serialization.json.buildJsonObject
import kotlinx.serialization.json.put

fun main() = runBlocking {
    val server = Server(
        serverInfo = Implementation(name = "Weather Service", version = "1.0.0"),
        options = ServerOptions(
            capabilities = ServerCapabilities(
                tools = ServerCapabilities.Tools(),
                // Advertise support for the content negotiation extension.
                // The empty object signals availability; feature tag parsing
                // is done on the client capabilities received at initialize.
                experimental = buildJsonObject {
                    put("io.modelcontextprotocol/content-negotiation", buildJsonObject { })
                }
            )
        )
    )

    server.addTool(
        name = "get_weather",
        description = "Return weather data for a location."
    ) { request ->
        // Access client capabilities via the ClientConnection receiver.
        // 'this.session.clientCapabilities' provides the capabilities captured
        // during the initialize handshake (verify visibility in your SDK version;
        // see Design Notes).
        val features = getFeatures(this.session.clientCapabilities)

        val location = request.arguments["location"]
            ?.let { (it as? kotlinx.serialization.json.JsonPrimitive)?.content }
            ?: "unknown"

        // Raw data — always computed the same way regardless of negotiation
        val data = mapOf(
            "location"                  to location,
            "temperature_c"             to 8,
            "humidity_percent"          to 72,
            "precipitation_probability" to 0.30,
            "wind_speed_kmh"            to 15,
            "uv_index"                  to 2
        )

        val text = if (features.isAgent && features.format == "json") {
            if (features.verbosity == "compact") {
                // Compact structured payload for agents
                Json.encodeToString(data)
            } else {
                // Verbose: add units and metadata
                Json.encodeToString(data + mapOf(
                    "units"             to "metric",
                    "source"            to "WeatherAPI",
                    "valid_for_minutes" to 60
                ))
            }
        } else {
            // Human-readable markdown
            buildList {
                add("## Weather in $location")
                add("- **Temperature**: ${data["temperature_c"]}°C")
                add("- **Humidity**: ${data["humidity_percent"]}%")
                add("- **Precipitation**: ${(0.30 * 100).toInt()}% chance")
                add("- **Wind**: ${data["wind_speed_kmh"]} km/h")
                if (features.verbosity == "verbose") {
                    add("- **UV Index**: ${data["uv_index"]} (low)")
                    add("")
                    add("_Data provided by WeatherAPI. Valid for 60 minutes._")
                }
            }.joinToString("\n")
        }

        io.modelcontextprotocol.kotlin.sdk.CallToolResult(
            content = listOf(TextContent(text))
        )
    }

    val transport = StdioServerTransport()
    server.connect(transport)
}
```

## Client Initialization Example

```kotlin
// ClientExample.kt — declares content negotiation at initialize
import io.modelcontextprotocol.kotlin.sdk.ClientCapabilities
import io.modelcontextprotocol.kotlin.sdk.client.Client
import io.modelcontextprotocol.kotlin.sdk.client.StdioClientTransport
import io.modelcontextprotocol.kotlin.sdk.Implementation
import kotlinx.serialization.json.buildJsonObject
import kotlinx.serialization.json.put
import kotlinx.serialization.json.putJsonArray

val client = Client(
    clientInfo = Implementation(name = "my-agent", version = "1.0.0"),
    capabilities = ClientCapabilities(
        experimental = buildJsonObject {
            put("io.modelcontextprotocol/content-negotiation", buildJsonObject {
                put("version", "1.0")
                putJsonArray("features") {
                    add(kotlinx.serialization.json.JsonPrimitive("agent"))
                    add(kotlinx.serialization.json.JsonPrimitive("sampling"))
                    add(kotlinx.serialization.json.JsonPrimitive("format=json"))
                    add(kotlinx.serialization.json.JsonPrimitive("verbosity=compact"))
                }
            })
        }
    )
)

val transport = StdioClientTransport(
    params = ServerParameters(command = "java", args = listOf("-jar", "weather-server.jar"))
)
client.connect(transport)

val result = client.callTool(
    name = "get_weather",
    arguments = mapOf("location" to "Bern")
)
println(result.content.first()) // → compact JSON string
```

## Design Notes

- **`ClientConnection` receiver**: `addTool` handlers use `ClientConnection` as
  their coroutine receiver. Client capabilities are accessible via
  `this.session.clientCapabilities`, where `session` holds the `ServerSession`
  established during `initialize`. Check your SDK version's visibility modifiers —
  if `session` is marked `internal`, access capabilities via an initialize
  lifecycle hook instead (same pattern as the [C# reference](CSHARP_REFERENCE.md)).
- **Session-scoped**: `clientCapabilities` is populated once during the
  `initialize` handshake and is the same for the lifetime of the connection,
  matching the session-scoped negotiation model.
- **`experimental` vs `extensions`**: The MCP Kotlin SDK maps `ClientCapabilities`
  from JSON using `kotlinx.serialization`. The `extensions` key in the JSON wire
  format lands in the `experimental: JsonObject?` property — both are open-ended
  property maps in the schema. `getFeatures` reads from `experimental` and
  documents this mapping explicitly.
- **Safe cast chain**: `experimental[EXTENSION_ID]` is cast with `as? JsonObject`
  and `ext["features"]` with `as? JsonArray`; the surrounding `try/catch` ensures
  a graceful fallback to `Features.EMPTY` on any unexpected shape.
- **Kotlin `data class`**: idiomatic Kotlin; immutable by default, structural
  equality, no boilerplate. Mirrors the Python `Features` Pydantic model,
  TypeScript `Features` class, and Java `Features` record.
- **`buildJsonObject {}` DSL**: idiomatic Kotlin for constructing `JsonObject`
  values without manual serialization.
- **Safe defaults**: all `Features` properties return sensible defaults
  (`format` → `"markdown"`, `verbosity` → `"standard"`) so tools degrade
  gracefully when no negotiation occurred.
- **Security**: feature tags only influence content shape — the tool always
  computes the same underlying data and only varies the presentation. Tags never
  control authentication or access decisions.
- **Extensible**: `Features.hasTag()` and `Features.vendorValue()` support vendor
  tags (`x-mycompany-hint=value`) without any changes to `getFeatures`.
