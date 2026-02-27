# Rust Reference Implementation

This document provides a Rust reference implementation of the
`io.modelcontextprotocol/content-negotiation` extension using the
[official MCP Rust SDK (`rmcp`)](https://github.com/modelcontextprotocol/rust-sdk).

## Overview

The implementation has three layers:

1. **`Format` and `Verbosity` enums** — idiomatic Rust replacements for the
   string literals used in the Python/TypeScript/Java references. Exhaustive
   `match` at call sites is enforced at compile time at zero cost.
2. **`Features`** — a struct holding the negotiated tag list, with convenience
   methods for all standard negotiation axes.
3. **`extract_features(caps)`** — a free function that parses feature tags from
   the `initialize` client capabilities JSON and returns a `Features` value.
   Stored in a `OnceLock<Features>` field on the server handler: set once during
   `initialize`, read lock-free on every subsequent tool call.

## `Cargo.toml`

```toml
[dependencies]
rmcp = { version = "0.1", features = ["server"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
```

## Implementation

```rust
// content_negotiation.rs

use serde_json::Value;

pub const EXTENSION_ID: &str = "io.modelcontextprotocol/content-negotiation";

/// Negotiated format preference.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum Format {
    #[default]
    Markdown,
    Text,
    Json,
}

/// Negotiated verbosity preference.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum Verbosity {
    Compact,
    #[default]
    Standard,
    Verbose,
}

/// Negotiated content preferences for the current MCP session.
#[derive(Debug, Clone, Default)]
pub struct Features {
    tags: Vec<String>,
}

impl Features {
    pub fn new(tags: Vec<String>) -> Self {
        Self { tags }
    }

    // --- Convenience methods ---

    pub fn is_agent(&self) -> bool {
        self.tags.iter().any(|t| t == "agent")
    }

    pub fn is_human(&self) -> bool {
        self.tags.iter().any(|t| t == "human")
    }

    pub fn is_interactive(&self) -> bool {
        self.tags.iter().any(|t| t == "interactive")
            && !self.tags.iter().any(|t| t == "!interactive")
    }

    pub fn has_sampling(&self) -> bool {
        self.tags.iter().any(|t| t == "sampling")
    }

    pub fn has_elicitation(&self) -> bool {
        self.tags.iter().any(|t| t == "elicitation")
    }

    pub fn format(&self) -> Format {
        for tag in &self.tags {
            if let Some(value) = tag.strip_prefix("format=") {
                match value {
                    "json" => return Format::Json,
                    "text" => return Format::Text,
                    "markdown" => return Format::Markdown,
                    _ => {}
                }
            }
        }
        Format::default() // Markdown
    }

    pub fn verbosity(&self) -> Verbosity {
        for tag in &self.tags {
            if let Some(value) = tag.strip_prefix("verbosity=") {
                match value {
                    "compact" => return Verbosity::Compact,
                    "standard" => return Verbosity::Standard,
                    "verbose" => return Verbosity::Verbose,
                    _ => {}
                }
            }
        }
        Verbosity::default() // Standard
    }

    /// Check for any feature tag, including vendor tags (`x-*`).
    pub fn has_tag(&self, tag: &str) -> bool {
        self.tags.iter().any(|t| t == tag)
    }

    /// Return the value of a `prefix=value` vendor tag, or `None`.
    ///
    /// Example: `features.vendor_value("x-mycompany-hint")` returns
    /// `Some("value")` when the tag `x-mycompany-hint=value` is present.
    pub fn vendor_value(&self, prefix: &str) -> Option<&str> {
        let needle = format!("{}=", prefix);
        self.tags
            .iter()
            .find(|t| t.starts_with(&needle))
            .map(|t| &t[needle.len()..])
    }
}

/// Extract [`Features`] from the client capabilities received at `initialize`.
///
/// Pass `serde_json::to_value(&init.capabilities)` (or a reference to the
/// capabilities `Value` if already serialised). Returns [`Features::default()`]
/// (all defaults) when no negotiation was declared by the client.
pub fn extract_features(capabilities: &Value) -> Features {
    let tags: Vec<String> = (|| {
        let extensions = capabilities.get("extensions")?.as_object()?;
        let ext = extensions.get(EXTENSION_ID)?.as_object()?;
        let features = ext.get("features")?.as_array()?;
        let tags = features
            .iter()
            .filter_map(|v| v.as_str())
            .map(String::from)
            .collect();
        Some(tags)
    })()
    .unwrap_or_default();

    Features::new(tags)
}
```

## Server Example

```rust
// main.rs

use std::sync::OnceLock;

use rmcp::{
    Error as McpError, RoleServer, ServerHandler,
    handler::server::tool::ToolCallContext,
    model::{
        ClientCapabilities, InitializeRequestParam, InitializeResult,
        ServerCapabilities, ServerInfo,
    },
    service::RequestContext,
    tool, serde_json,
};

use content_negotiation::{extract_features, Format, Verbosity, Features};

struct WeatherServer {
    features: OnceLock<Features>,
}

impl WeatherServer {
    fn new() -> Self {
        Self {
            features: OnceLock::new(),
        }
    }

    /// Return the negotiated features for this session.
    ///
    /// Falls back to [`Features::default()`] (all defaults) before `initialize`
    /// completes or when no negotiation was declared by the client.
    fn features(&self) -> Features {
        self.features.get().cloned().unwrap_or_default()
    }
}

#[rmcp::async_trait]
impl ServerHandler for WeatherServer {
    fn get_info(&self) -> ServerInfo {
        ServerInfo {
            name: "Weather Service".into(),
            version: "1.0.0".into(),
        }
    }

    fn get_capabilities(&self, _context: &RequestContext<RoleServer>) -> ServerCapabilities {
        ServerCapabilities {
            tools: Some(Default::default()),
            // Advertise support for the content negotiation extension.
            // The empty object signals availability; feature tag parsing
            // is done on the client capabilities received at initialize.
            experimental: Some(serde_json::json!({
                "io.modelcontextprotocol/content-negotiation": {}
            })),
            ..Default::default()
        }
    }

    async fn initialize(
        &self,
        init: InitializeRequestParam,
        context: RequestContext<RoleServer>,
    ) -> Result<InitializeResult, McpError> {
        // Parse and store feature tags — runs exactly once per connection.
        let caps_json = serde_json::to_value(&init.capabilities)
            .unwrap_or(serde_json::Value::Null);
        let features = extract_features(&caps_json);
        let _ = self.features.set(features); // OnceLock: silent no-op on repeat

        // Delegate to the default implementation for the rest.
        rmcp::handler::server::default_initialize(self, init, context).await
    }
}

#[tool(tool_box)]
impl WeatherServer {
    /// Return weather data for a location.
    #[tool(description = "Return weather data for a location.")]
    async fn get_weather(&self, location: String) -> String {
        let features = self.features();

        // Raw data — always computed the same way regardless of negotiation
        let temp_c = 8;
        let humidity = 72;
        let precip = 0.30_f64;
        let wind = 15;
        let uv = 2;

        if features.is_agent() && features.format() == Format::Json {
            return match features.verbosity() {
                Verbosity::Compact => serde_json::json!({
                    "location": location,
                    "temperature_c": temp_c,
                    "humidity_percent": humidity,
                    "precipitation_probability": precip,
                    "wind_speed_kmh": wind,
                    "uv_index": uv,
                })
                .to_string(),
                _ => serde_json::json!({
                    "location": location,
                    "temperature_c": temp_c,
                    "humidity_percent": humidity,
                    "precipitation_probability": precip,
                    "wind_speed_kmh": wind,
                    "uv_index": uv,
                    "units": "metric",
                    "source": "WeatherAPI",
                    "valid_for_minutes": 60,
                })
                .to_string(),
            };
        }

        // Human-readable markdown
        let mut lines = vec![
            format!("## Weather in {location}"),
            format!("- **Temperature**: {temp_c}°C"),
            format!("- **Humidity**: {humidity}%"),
            format!("- **Precipitation**: {}% chance", (precip * 100.0) as u32),
            format!("- **Wind**: {wind} km/h"),
        ];
        if features.verbosity() == Verbosity::Verbose {
            lines.push(format!("- **UV Index**: {uv} (low)"));
            lines.push(String::new());
            lines.push("_Data provided by WeatherAPI. Valid for 60 minutes._".into());
        }
        lines.join("\n")
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let server = WeatherServer::new();
    let transport = rmcp::transport::stdio();
    server.serve(transport).await?.waiting().await?;
    Ok(())
}
```

## Client Initialization Example

```rust
// client_example.rs — declares content negotiation at initialize
use rmcp::{
    RoleClient, ServiceExt,
    model::{CallToolRequestParam, ClientCapabilities, ClientInfo},
    transport::stdio,
};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let transport = stdio(); // or any other rmcp transport

    let client_info = ClientInfo {
        name: "my-agent".into(),
        version: "1.0.0".into(),
    };

    // Declare content negotiation features in the experimental capabilities map.
    // The MCP Rust SDK serialises ClientCapabilities to JSON; the extensions key
    // lands in the wire-format capabilities.extensions object as specified.
    let capabilities = ClientCapabilities {
        experimental: Some(serde_json::json!({
            "extensions": {
                "io.modelcontextprotocol/content-negotiation": {
                    "version": "1.0",
                    "features": [
                        "agent",
                        "sampling",
                        "format=json",
                        "verbosity=compact"
                    ]
                }
            }
        })),
        ..Default::default()
    };

    let client = client_info
        .serve_with_capabilities(transport, capabilities)
        .await?;

    let result = client
        .call_tool(CallToolRequestParam {
            name: "get_weather".into(),
            arguments: Some(serde_json::json!({ "location": "Bern" })),
        })
        .await?;

    println!("{result:?}"); // → compact JSON string
    Ok(())
}
```

## Design Notes

- **Enums over strings**: `Format` and `Verbosity` are proper Rust enums rather
  than `&str` returns. `match features.format() { Format::Json => … }` is
  exhaustive at compile time — the compiler rejects unhandled variants. Adding a
  new format variant surfaces every affected call site immediately.
- **`OnceLock` for session scope**: `initialize` is called exactly once per
  connection. `OnceLock::set` succeeds on that first call; subsequent calls
  (which cannot happen in normal MCP operation) are silently ignored. All later
  `get()` calls are lock-free reads — no `Mutex` overhead on the hot path.
- **Thread safety**: `rmcp` wraps handlers in `Arc<T>`. `OnceLock<T>` is
  `Sync` when `T: Send + Sync`, so no additional synchronisation is needed.
  `Features` derives neither `Send` nor `Sync` explicitly, but they are
  inferred automatically because `Vec<String>` is both.
- **`strip_prefix`**: idiomatic Rust alternative to manual `starts_with` +
  slice indexing for parsing `format=` and `verbosity=` key-value tags.
- **Closure-based `extract_features`**: the `(|| { … })().unwrap_or_default()`
  pattern chains `Option`-returning operations cleanly without nested `if let`
  or `match` ladders. Any missing or malformed capability silently yields
  `Features::default()`.
- **Safe defaults**: `#[default]` on `Format::Markdown` and
  `Verbosity::Standard` means `Format::default()` and `Verbosity::default()`
  return the correct fallbacks. `Features::default()` has an empty tag list,
  so `features()` on an uninitialised handler is safe.
- **Security**: feature tags only influence content shape — the tool always
  computes the same underlying data and only varies the presentation. Tags
  never control authentication or access decisions.
- **Extensible**: `Features::has_tag()` and `Features::vendor_value()` support
  vendor tags (`x-mycompany-hint=value`) without any changes to
  `extract_features` or the middleware layer.
