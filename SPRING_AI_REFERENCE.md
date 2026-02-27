# Spring AI 2.0 Reference Implementation

This document provides a Java reference implementation of the
`io.modelcontextprotocol/content-negotiation` extension using
[Spring AI 2.0](https://docs.spring.io/spring-ai/reference/) and its MCP server support.

## Overview

The implementation has two layers:

1. **`Features`** — a Java 17 `record` with convenience methods for the most common
   negotiation axes, mirroring the Python `Features` Pydantic model and TypeScript `Features`
   class.
2. **`ContentNegotiation.getFeatures(exchange)`** — a static helper that reads client
   capabilities from the `McpSyncServerExchange` injected by Spring AI into every `@Tool`
   method. No middleware, no thread-locals; capabilities are available directly on the
   exchange object.

## Dependencies

### Maven

```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-starter-mcp-server</artifactId>
</dependency>
```

For SSE / Streamable HTTP transport instead of STDIO, replace with:

```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
</dependency>
```

### Gradle

```groovy
implementation 'org.springframework.ai:spring-ai-starter-mcp-server'
```

## Implementation

```java
// ContentNegotiation.java
package com.example.mcp;

import io.modelcontextprotocol.server.McpSyncServerExchange;

import java.util.List;
import java.util.Map;
import java.util.Optional;

public final class ContentNegotiation {

    private static final String EXTENSION_ID =
            "io.modelcontextprotocol/content-negotiation";

    private ContentNegotiation() {}

    /**
     * Negotiated content preferences for the current MCP session.
     *
     * <p>Constructed once per tool invocation from the client capabilities
     * captured during {@code initialize}. All accessors have safe defaults so
     * tools degrade gracefully when no negotiation occurred.
     */
    public record Features(List<String> tags) {

        /** Sentinel returned when no negotiation took place. */
        public static final Features EMPTY = new Features(List.of());

        /** {@code true} when the client declared the {@code agent} tag. */
        public boolean isAgent() {
            return tags.contains("agent");
        }

        /** {@code true} when the client declared the {@code human} tag. */
        public boolean isHuman() {
            return tags.contains("human");
        }

        /**
         * {@code true} when {@code interactive} is present and
         * {@code !interactive} is absent.
         */
        public boolean isInteractive() {
            return tags.contains("interactive") && !tags.contains("!interactive");
        }

        /** {@code true} when the client declared the {@code sampling} tag. */
        public boolean hasSampling() {
            return tags.contains("sampling");
        }

        /** {@code true} when the client declared the {@code elicitation} tag. */
        public boolean hasElicitation() {
            return tags.contains("elicitation");
        }

        /**
         * Returns the negotiated format: {@code "json"}, {@code "text"}, or
         * {@code "markdown"} (default).
         */
        public String format() {
            return tags.stream()
                    .filter(t -> t.startsWith("format="))
                    .map(t -> t.substring("format=".length()))
                    .filter(v -> v.equals("json") || v.equals("text") || v.equals("markdown"))
                    .findFirst()
                    .orElse("markdown");
        }

        /**
         * Returns the negotiated verbosity: {@code "compact"}, {@code "standard"}
         * (default), or {@code "verbose"}.
         */
        public String verbosity() {
            return tags.stream()
                    .filter(t -> t.startsWith("verbosity="))
                    .map(t -> t.substring("verbosity=".length()))
                    .filter(v -> v.equals("compact") || v.equals("standard") || v.equals("verbose"))
                    .findFirst()
                    .orElse("standard");
        }

        /** Check for any feature tag, including vendor tags ({@code x-*}). */
        public boolean hasTag(String tag) {
            return tags.contains(tag);
        }

        /**
         * Return the value of a {@code prefix=value} vendor tag, or empty.
         *
         * <p>Example: {@code features.vendorValue("x-mycompany-hint")}
         * returns {@code Optional.of("value")} when the tag
         * {@code x-mycompany-hint=value} is present.
         */
        public Optional<String> vendorValue(String prefix) {
            return tags.stream()
                    .filter(t -> t.startsWith(prefix + "="))
                    .map(t -> t.substring(prefix.length() + 1))
                    .findFirst();
        }
    }

    /**
     * Return the negotiated {@link Features} for the current session.
     *
     * <p>Falls back to {@link Features#EMPTY} (all defaults) when no negotiation
     * took place or the extension was not declared by the client.
     *
     * <p><b>Note on {@code experimental} vs {@code extensions}</b>: The MCP Java
     * SDK maps {@code ClientCapabilities} from JSON. The {@code extensions} key in
     * the JSON wire format lands in the {@code experimental} field of the Java
     * {@code ClientCapabilities} POJO — both are treated as
     * {@code additionalProperties}-style maps in the schema. This method reads the
     * extension from {@code experimental} accordingly.
     *
     * @param exchange the {@code McpSyncServerExchange} injected by Spring AI into
     *                 every {@code @Tool} method
     * @return negotiated features, never {@code null}
     */
    public static Features getFeatures(McpSyncServerExchange exchange) {
        try {
            var caps = exchange.getClientCapabilities();
            if (caps == null || caps.experimental() == null) return Features.EMPTY;

            Object ext = caps.experimental()
                             .get(EXTENSION_ID);
            if (!(ext instanceof Map<?, ?> extMap)) return Features.EMPTY;

            Object featuresObj = extMap.get("features");
            if (!(featuresObj instanceof List<?> rawList)) return Features.EMPTY;

            List<String> tags = rawList.stream()
                    .filter(String.class::isInstance)
                    .map(String.class::cast)
                    .toList();
            return new Features(tags);
        } catch (Exception e) {
            return Features.EMPTY;
        }
    }
}
```

## Server Setup

### `application.properties`

```properties
spring.ai.mcp.server.name=Weather Service
spring.ai.mcp.server.version=1.0.0
```

### Capability Advertisement

Advertise support for the extension via a `McpServerFeatures.SyncSpec` customizer bean.
The empty object signals availability; feature tag parsing is done on the client
capabilities received at `initialize`.

```java
// McpConfig.java
package com.example.mcp;

import io.modelcontextprotocol.server.McpServerFeatures;
import io.modelcontextprotocol.spec.McpSchema.ServerCapabilities;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.Map;

@Configuration
public class McpConfig {

    @Bean
    McpServerFeatures.SyncSpec mcpFeatures() {
        return spec -> spec.capabilities(
            ServerCapabilities.builder()
                .tools(true)
                .experimental(Map.of(
                    "io.modelcontextprotocol/content-negotiation", Map.of()
                ))
                .build()
        );
    }
}
```

## Server Example

```java
// WeatherTools.java
package com.example.mcp;

import io.modelcontextprotocol.server.McpSyncServerExchange;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Component;

import com.example.mcp.ContentNegotiation.Features;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

@Component
public class WeatherTools {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Tool(name = "get_weather", description = "Return weather data for a location.")
    public String getWeather(
            @ToolParam(description = "City or region name") String location,
            McpSyncServerExchange exchange) {

        Features features = ContentNegotiation.getFeatures(exchange);

        // Raw data — always computed the same way regardless of negotiation
        var data = new LinkedHashMap<String, Object>();
        data.put("location", location);
        data.put("temperature_c", 8);
        data.put("humidity_percent", 72);
        data.put("precipitation_probability", 0.30);
        data.put("wind_speed_kmh", 15);
        data.put("uv_index", 2);

        if (features.isAgent() && "json".equals(features.format())) {
            if ("compact".equals(features.verbosity())) {
                // Compact structured payload for agents
                return toJson(data);
            }
            // Verbose: add units and metadata
            var verbose = new LinkedHashMap<>(data);
            verbose.put("units", "metric");
            verbose.put("source", "WeatherAPI");
            verbose.put("valid_for_minutes", 60);
            return toJson(verbose);
        }

        // Human-readable markdown
        var lines = new ArrayList<>(List.of(
            "## Weather in " + location,
            "- **Temperature**: " + data.get("temperature_c") + "°C",
            "- **Humidity**: " + data.get("humidity_percent") + "%",
            "- **Precipitation**: " + (int)(0.30 * 100) + "% chance",
            "- **Wind**: " + data.get("wind_speed_kmh") + " km/h"
        ));
        if ("verbose".equals(features.verbosity())) {
            lines.add("- **UV Index**: " + data.get("uv_index") + " (low)");
            lines.add("");
            lines.add("_Data provided by WeatherAPI. Valid for 60 minutes._");
        }
        return String.join("\n", lines);
    }

    private String toJson(Object value) {
        try {
            return objectMapper.writeValueAsString(value);
        } catch (Exception e) {
            return value.toString();
        }
    }
}
```

### Application Entry Point

```java
// WeatherServiceApplication.java
package com.example.mcp;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class WeatherServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(WeatherServiceApplication.class, args);
    }
}
```

## Client Initialization Example

```java
// ClientExample.java — declares content negotiation at initialize
import io.modelcontextprotocol.client.McpClient;
import io.modelcontextprotocol.spec.McpSchema.ClientCapabilities;

var client = McpClient.sync(transport)
    .clientInfo(new McpSchema.Implementation("my-agent", "1.0.0"))
    .capabilities(ClientCapabilities.builder()
        .experimental(Map.of(
            "io.modelcontextprotocol/content-negotiation", Map.of(
                "version", "1.0",
                "features", List.of(
                    "agent",
                    "sampling",
                    "format=json",
                    "verbosity=compact"
                )
            )
        ))
        .build())
    .build();

var result = client.callTool(
    new McpSchema.CallToolRequest("get_weather", Map.of("location", "Bern"))
);
System.out.println(result); // → compact JSON string
```

## Design Notes

- **`McpSyncServerExchange` as special parameter**: Spring AI's annotation processor
  detects `McpSyncServerExchange` as a parameter on any `@Tool` method and injects it
  automatically. No middleware, no thread-locals — simpler than the Python approach and
  on par with TypeScript.
- **Session-scoped**: `exchange.getClientCapabilities()` returns the capabilities
  captured during the `initialize` handshake; they are the same for the lifetime of the
  connection, matching the session-scoped negotiation model.
- **`experimental` vs `extensions`**: The MCP Java SDK's `ClientCapabilities` POJO has
  an `experimental` field of type `Map<String, Object>`. When a client sends
  `capabilities.extensions` in the JSON, the SDK deserialises it into `experimental`
  (both fields are open-ended property maps in the schema). `getFeatures` reads from
  `experimental` and documents this mapping explicitly.
- **Safe cast pattern**: the value retrieved from `experimental` is typed as
  `Object` at runtime. The `instanceof Map<?,?>` and `instanceof List<?>` pattern
  matches (Java 16+) provide type-safe narrowing without unchecked casts; the
  surrounding `try/catch` ensures a graceful fallback to `Features.EMPTY` on any
  unexpected shape.
- **Java `record`**: idiomatic Java 17+; immutable, no boilerplate. Mirrors the
  Python `Features` Pydantic model and TypeScript `Features` class exactly in behaviour.
- **Safe defaults**: all `Features` accessors have sensible defaults
  (`format()` → `"markdown"`, `verbosity()` → `"standard"`) so tools degrade
  gracefully when no negotiation occurred.
- **Security**: feature tags only influence content shape — the tool always computes
  the same underlying data and only varies the presentation. Tags never control
  authentication or access decisions.
- **Extensible**: `Features.hasTag()` and `Features.vendorValue()` support vendor
  tags (`x-mycompany-hint=value`) without any changes to `getFeatures`.
