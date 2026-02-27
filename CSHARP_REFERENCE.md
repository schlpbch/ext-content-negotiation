# C# / .NET Reference Implementation

This document provides a C# reference implementation of the
`io.modelcontextprotocol/content-negotiation` extension using the
[official MCP .NET SDK](https://github.com/modelcontextprotocol/csharp-sdk)
(`ModelContextProtocol` NuGet package, v1.0+).

## Overview

The implementation has three layers:

1. **`Features`** — a C# `record` with convenience properties for the most
   common negotiation axes, mirroring the Python `Features` Pydantic model and
   TypeScript `Features` class.
2. **`ContentNegotiation.ParseFeatures(ClientCapabilities?)`** — parses feature
   tags from the client capabilities received at `initialize`.
3. **`ContentNegotiationState`** — a singleton service that captures the
   negotiated `Features` per session during the initialize handshake and makes
   them available inside `[McpServerTool]` methods via dependency injection.

> **Why the singleton?** The `[McpServerTool]` method receives a `McpServer`
> parameter with `SessionId` and `NegotiatedProtocolVersion`, but `ClientCapabilities`
> are not directly exposed on it. Capturing them in an initialize handler and
> storing them by session ID is the idiomatic .NET pattern.

## Dependencies

```xml
<!-- .csproj -->
<PackageReference Include="ModelContextProtocol" Version="1.0.0" />
```

```bash
dotnet add package ModelContextProtocol
```

For SSE / HTTP transport instead of STDIO, also add:

```xml
<PackageReference Include="ModelContextProtocol.AspNetCore" Version="1.0.0" />
```

## Implementation

```csharp
// ContentNegotiation.cs
using System.Text.Json;
using ModelContextProtocol.Protocol;

namespace WeatherService;

/// <summary>
/// Negotiated content preferences for the current MCP session.
/// All properties return safe defaults when no negotiation took place.
/// </summary>
public sealed record Features(IReadOnlyList<string> Tags)
{
    /// <summary>Sentinel returned when no negotiation took place.</summary>
    public static readonly Features Empty = new(Array.Empty<string>());

    /// <summary>True when the client declared the <c>agent</c> tag.</summary>
    public bool IsAgent => Tags.Contains("agent");

    /// <summary>True when the client declared the <c>human</c> tag.</summary>
    public bool IsHuman => Tags.Contains("human");

    /// <summary>
    /// True when <c>interactive</c> is present and <c>!interactive</c> is absent.
    /// </summary>
    public bool IsInteractive =>
        Tags.Contains("interactive") && !Tags.Contains("!interactive");

    /// <summary>True when the client declared the <c>sampling</c> tag.</summary>
    public bool HasSampling => Tags.Contains("sampling");

    /// <summary>True when the client declared the <c>elicitation</c> tag.</summary>
    public bool HasElicitation => Tags.Contains("elicitation");

    /// <summary>
    /// Returns the negotiated format: <c>"json"</c>, <c>"text"</c>,
    /// or <c>"markdown"</c> (default).
    /// </summary>
    public string Format
    {
        get
        {
            foreach (var tag in Tags)
            {
                if (tag.StartsWith("format=", StringComparison.Ordinal))
                {
                    var v = tag["format=".Length..];
                    if (v is "json" or "text" or "markdown") return v;
                }
            }
            return "markdown";
        }
    }

    /// <summary>
    /// Returns the negotiated verbosity: <c>"compact"</c>,
    /// <c>"standard"</c> (default), or <c>"verbose"</c>.
    /// </summary>
    public string Verbosity
    {
        get
        {
            foreach (var tag in Tags)
            {
                if (tag.StartsWith("verbosity=", StringComparison.Ordinal))
                {
                    var v = tag["verbosity=".Length..];
                    if (v is "compact" or "standard" or "verbose") return v;
                }
            }
            return "standard";
        }
    }

    /// <summary>Check for any feature tag, including vendor tags (<c>x-*</c>).</summary>
    public bool HasTag(string tag) => Tags.Contains(tag);

    /// <summary>
    /// Return the value of a <c>prefix=value</c> vendor tag, or <c>null</c>.
    /// <para>
    /// Example: <c>features.VendorValue("x-mycompany-hint")</c> returns
    /// <c>"value"</c> when the tag <c>x-mycompany-hint=value</c> is present.
    /// </para>
    /// </summary>
    public string? VendorValue(string prefix)
    {
        var key = prefix + "=";
        foreach (var tag in Tags)
            if (tag.StartsWith(key, StringComparison.Ordinal))
                return tag[key.Length..];
        return null;
    }
}

/// <summary>
/// Parses content negotiation feature tags from MCP client capabilities.
/// </summary>
public static class ContentNegotiation
{
    private const string ExtensionId = "io.modelcontextprotocol/content-negotiation";

    /// <summary>
    /// Parse feature tags from <paramref name="capabilities"/>.
    /// Returns <see cref="Features.Empty"/> on any missing or malformed data.
    ///
    /// <para>
    /// <b>Note on <c>Experimental</c> vs <c>Extensions</c></b>: The MCP .NET SDK
    /// maps <c>ClientCapabilities</c> from JSON. The <c>extensions</c> key in the
    /// JSON wire format lands in the <c>Experimental</c> property of the C# POJO
    /// (both are treated as open-ended property maps in the schema). This method
    /// reads the extension from <c>Experimental</c> accordingly.
    /// </para>
    /// </summary>
    public static Features ParseFeatures(ClientCapabilities? capabilities)
    {
        try
        {
            if (capabilities?.Experimental is not { } experimental)
                return Features.Empty;

            if (!experimental.TryGetValue(ExtensionId, out var extElement))
                return Features.Empty;

            if (extElement.ValueKind != JsonValueKind.Object)
                return Features.Empty;

            if (!extElement.TryGetProperty("features", out var featuresElement))
                return Features.Empty;

            if (featuresElement.ValueKind != JsonValueKind.Array)
                return Features.Empty;

            var tags = featuresElement.EnumerateArray()
                .Where(e => e.ValueKind == JsonValueKind.String)
                .Select(e => e.GetString()!)
                .ToList();

            return new Features(tags);
        }
        catch
        {
            return Features.Empty;
        }
    }
}

/// <summary>
/// Singleton service that holds the negotiated <see cref="Features"/> for each
/// active MCP session, keyed by session ID.
///
/// <para>
/// Populated during the <c>initialize</c> handshake via a registered handler;
/// consumed inside <c>[McpServerTool]</c> methods via dependency injection.
/// For single-session transports (STDIO) the session ID may be <c>null</c>,
/// in which case a single slot is used.
/// </para>
/// </summary>
public sealed class ContentNegotiationState
{
    private readonly ConcurrentDictionary<string, Features> _sessions = new();
    private Features _defaultFeatures = Features.Empty;

    internal void Set(string? sessionId, Features features)
    {
        if (sessionId is null)
            _defaultFeatures = features;
        else
            _sessions[sessionId] = features;
    }

    /// <summary>
    /// Returns the negotiated <see cref="Features"/> for the given session,
    /// or <see cref="Features.Empty"/> if none were negotiated.
    /// </summary>
    public Features Get(string? sessionId) =>
        sessionId is not null && _sessions.TryGetValue(sessionId, out var f)
            ? f
            : _defaultFeatures;
}
```

## Server Setup

```csharp
// Program.cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using ModelContextProtocol;
using WeatherService;

// Use CreateEmptyApplicationBuilder to prevent console output from corrupting
// the STDIO JSON-RPC stream.
var builder = Host.CreateEmptyApplicationBuilder(settings: null);

builder.Services.AddSingleton<ContentNegotiationState>();

builder.Services
    .AddMcpServer(options =>
    {
        options.ServerInfo = new Implementation
        {
            Name    = "Weather Service",
            Version = "1.0.0",
        };
    })
    .WithStdioServerTransport()
    .WithTools<WeatherTools>()
    // Capture client capabilities at initialize and store them for tool use.
    .WithInitializedHandler(async (ctx, ct) =>
    {
        var state = ctx.Services.GetRequiredService<ContentNegotiationState>();
        var features = ContentNegotiation.ParseFeatures(
            ctx.Params?.Capabilities);
        state.Set(ctx.Server.SessionId, features);
    });

await builder.Build().RunAsync();
```

> **Advertising the extension**: To signal support for content negotiation in the
> server's `initialize` response, set the extension key in server experimental
> capabilities:
>
> ```csharp
> options.Capabilities = new ServerCapabilities
> {
>     Experimental = new Dictionary<string, JsonElement>
>     {
>         ["io.modelcontextprotocol/content-negotiation"] =
>             JsonSerializer.SerializeToElement(new { }),
>     },
> };
> ```

## Server Example

```csharp
// WeatherTools.cs
using System.ComponentModel;
using System.Text.Json;
using ModelContextProtocol.Server;
using WeatherService;

[McpServerToolType]
public static class WeatherTools
{
    [McpServerTool(Name = "get_weather")]
    [Description("Return weather data for a location.")]
    public static string GetWeather(
        McpServer server,
        ContentNegotiationState state,
        [Description("City or region name")] string location)
    {
        var features = state.Get(server.SessionId);

        // Raw data — always computed the same way regardless of negotiation
        var data = new Dictionary<string, object>
        {
            ["location"]                  = location,
            ["temperature_c"]             = 8,
            ["humidity_percent"]          = 72,
            ["precipitation_probability"] = 0.30,
            ["wind_speed_kmh"]            = 15,
            ["uv_index"]                  = 2,
        };

        if (features.IsAgent && features.Format == "json")
        {
            if (features.Verbosity == "compact")
            {
                // Compact structured payload for agents
                return JsonSerializer.Serialize(data);
            }
            // Verbose: add units and metadata
            var verbose = new Dictionary<string, object>(data)
            {
                ["units"]             = "metric",
                ["source"]            = "WeatherAPI",
                ["valid_for_minutes"] = 60,
            };
            return JsonSerializer.Serialize(verbose);
        }

        // Human-readable markdown
        var lines = new List<string>
        {
            $"## Weather in {location}",
            $"- **Temperature**: {data["temperature_c"]}°C",
            $"- **Humidity**: {data["humidity_percent"]}%",
            $"- **Precipitation**: {(int)(0.30 * 100)}% chance",
            $"- **Wind**: {data["wind_speed_kmh"]} km/h",
        };
        if (features.Verbosity == "verbose")
        {
            lines.Add($"- **UV Index**: {data["uv_index"]} (low)");
            lines.Add("");
            lines.Add("_Data provided by WeatherAPI. Valid for 60 minutes._");
        }
        return string.Join("\n", lines);
    }
}
```

## Client Initialization Example

```csharp
// ClientExample.cs — declares content negotiation at initialize
using ModelContextProtocol.Client;
using ModelContextProtocol.Protocol;
using System.Text.Json;

var clientCapabilities = new ClientCapabilities
{
    Experimental = new Dictionary<string, JsonElement>
    {
        ["io.modelcontextprotocol/content-negotiation"] =
            JsonSerializer.SerializeToElement(new
            {
                version  = "1.0",
                features = new[] { "agent", "sampling", "format=json", "verbosity=compact" },
            }),
    },
};

await using var transport = new StdioClientTransport(new StdioClientTransportOptions
{
    Command = "dotnet",
    Arguments = ["run", "--project", "WeatherService"],
});

await using var client = await McpClientFactory.CreateAsync(
    transport,
    new McpClientOptions
    {
        ClientInfo    = new Implementation { Name = "my-agent", Version = "1.0.0" },
        Capabilities  = clientCapabilities,
    });

var result = await client.CallToolAsync(
    "get_weather",
    new Dictionary<string, object?> { ["location"] = "Bern" });

Console.WriteLine(result.Content[0].Text); // → compact JSON string
```

## Design Notes

- **`ContentNegotiationState` singleton**: The `[McpServerTool]` method receives
  `McpServer` (with `SessionId`) but not `ClientCapabilities` directly. Capturing
  capabilities in an initialize handler and storing them in a DI-registered singleton
  is the idiomatic .NET pattern. Both STDIO (single session, null `SessionId`) and
  HTTP/SSE (multiple sessions, non-null `SessionId`) are handled.
- **`Experimental` vs `Extensions`**: The MCP .NET SDK maps `ClientCapabilities`
  from JSON. The `extensions` key in the JSON wire format lands in the
  `Experimental` property of the C# POJO — both are open-ended property maps in
  the schema. `ParseFeatures` reads from `Experimental` and documents this mapping
  explicitly.
- **`JsonElement` safe traversal**: `Experimental` values are typed as
  `JsonElement`. Each step checks `ValueKind` before accessing properties;
  the surrounding `try/catch` ensures a graceful fallback to `Features.Empty` on
  any unexpected shape.
- **C# `record`**: Idiomatic C# 9+; immutable value semantics, no boilerplate.
  Mirrors the Python `Features` Pydantic model and TypeScript `Features` class.
- **Safe defaults**: all `Features` properties return sensible defaults
  (`Format` → `"markdown"`, `Verbosity` → `"standard"`) so tools degrade
  gracefully when no negotiation occurred.
- **Security**: feature tags only influence content shape — the tool always
  computes the same underlying data and only varies the presentation. Tags never
  control authentication or access decisions.
- **Extensible**: `Features.HasTag()` and `Features.VendorValue()` support vendor
  tags (`x-mycompany-hint=value`) without any changes to `ParseFeatures`.
- **STDIO logging**: never write to `Console.Out` in a STDIO server — it corrupts
  the JSON-RPC stream. Use `Console.Error` or `ILogger` for diagnostics.
