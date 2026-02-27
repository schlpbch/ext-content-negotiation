# FastMCP Reference Implementation

This document provides a Python reference implementation of the
`io.modelcontextprotocol/content-negotiation` extension using
[FastMCP](https://github.com/jlowin/fastmcp) (v3+).

## Overview

The implementation has three layers:

1. **`ContentNegotiationMiddleware`** — captures feature tags from the
   `initialize` handshake and stores them in session state.
2. **`Features`** — a Pydantic `BaseModel` with convenience properties for the most
   common negotiation axes.
3. **`get_features(ctx)`** — retrieves the negotiated `Features` from any
   tool or resource handler.

## Implementation

```python
# content_negotiation.py
from __future__ import annotations

from typing import Any, Literal

from fastmcp.server import Context
from fastmcp.server.middleware import CallNext, Middleware, MiddlewareContext
import mcp.types as mt
from pydantic import BaseModel, Field

EXTENSION_ID = "io.modelcontextprotocol/content-negotiation"
_STATE_KEY = "__content_negotiation_features__"


class Features(BaseModel):
    """Negotiated content preferences for the current session."""

    tags: list[str] = Field(default_factory=list)

    # --- Convenience properties ---

    @property
    def is_agent(self) -> bool:
        return "agent" in self.tags

    @property
    def is_human(self) -> bool:
        return "human" in self.tags

    @property
    def is_interactive(self) -> bool:
        return "interactive" in self.tags and "!interactive" not in self.tags

    @property
    def has_sampling(self) -> bool:
        return "sampling" in self.tags

    @property
    def has_elicitation(self) -> bool:
        return "elicitation" in self.tags

    @property
    def format(self) -> Literal["json", "text", "markdown"]:
        for tag in self.tags:
            if tag.startswith("format="):
                value = tag.split("=", 1)[1]
                if value in ("json", "text", "markdown"):
                    return value  # type: ignore[return-value]
        return "markdown"  # default

    @property
    def verbosity(self) -> Literal["compact", "standard", "verbose"]:
        for tag in self.tags:
            if tag.startswith("verbosity="):
                value = tag.split("=", 1)[1]
                if value in ("compact", "standard", "verbose"):
                    return value  # type: ignore[return-value]
        return "standard"  # default

    def has_tag(self, tag: str) -> bool:
        """Check for any feature tag, including vendor tags (x-*)."""
        return tag in self.tags

    def vendor_value(self, prefix: str) -> str | None:
        """Return the value of a key=value vendor tag, or None."""
        for tag in self.tags:
            if tag.startswith(f"{prefix}="):
                return tag.split("=", 1)[1]
        return None


class ContentNegotiationMiddleware(Middleware):
    """Captures content negotiation feature tags at initialize and stores
    them in session state for later retrieval by tool/resource handlers."""

    async def on_initialize(
        self,
        context: MiddlewareContext[mt.InitializeRequest],
        call_next: CallNext[mt.InitializeRequest, mt.InitializeResult | None],
    ) -> mt.InitializeResult | None:
        result = await call_next(context)

        # Parse feature tags from client capabilities
        tags: list[str] = []
        try:
            params: Any = context.message.params
            extensions: Any = (
                params.capabilities.model_extra.get("extensions", {})
                if params.capabilities.model_extra
                else {}
            )
            ext = extensions.get(EXTENSION_ID, {})
            tags = ext.get("features", [])
        except (AttributeError, TypeError):
            pass

        await context.fastmcp_context.set_state(_STATE_KEY, tags)
        return result


async def get_features(ctx: Context) -> Features:
    """Return the negotiated Features for the current session.

    Falls back to an empty Features (all defaults) when no negotiation
    took place or the extension was not declared by the client.
    """
    tags: list[str] = await ctx.get_state(_STATE_KEY) or []
    return Features(tags=tags)
```

## Server Example

```python
# server.py
import json
from fastmcp import FastMCP
from fastmcp.server import Context
from content_negotiation import ContentNegotiationMiddleware, get_features

mcp = FastMCP(
    "Weather Service",
    middleware=[ContentNegotiationMiddleware()],
)


@mcp.tool
async def get_weather(location: str, ctx: Context) -> str | dict:
    """Return weather data for a location."""
    features = await get_features(ctx)

    # Raw data — always computed the same way
    data = {
        "location": location,
        "temperature_c": 8,
        "humidity_percent": 72,
        "precipitation_probability": 0.30,
        "wind_speed_kmh": 15,
        "uv_index": 2,
    }

    if features.is_agent and features.format == "json":
        # Compact structured payload for agents
        if features.verbosity == "compact":
            return data
        # Verbose: add units and metadata
        return {**data, "units": "metric", "source": "WeatherAPI", "valid_for_minutes": 60}

    # Human-readable markdown
    lines = [
        f"## Weather in {location}",
        f"- **Temperature**: {data['temperature_c']}°C",
        f"- **Humidity**: {data['humidity_percent']}%",
        f"- **Precipitation**: {data['precipitation_probability']:.0%} chance",
        f"- **Wind**: {data['wind_speed_kmh']} km/h",
    ]
    if features.verbosity == "verbose":
        lines += [
            f"- **UV Index**: {data['uv_index']} (low)",
            "",
            "_Data provided by WeatherAPI. Valid for 60 minutes._",
        ]
    return "\n".join(lines)


if __name__ == "__main__":
    mcp.run()
```

## Client Initialization Example

```python
# client_example.py — declares content negotiation at initialize
from mcp import ClientSession
from mcp.client.stdio import stdio_client, StdioServerParameters

params = StdioServerParameters(command="python", args=["server.py"])

async with stdio_client(params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize(
            experimental_capabilities={
                "extensions": {
                    "io.modelcontextprotocol/content-negotiation": {
                        "version": "1.0",
                        "features": [
                            "agent",
                            "sampling",
                            "format=json",
                            "verbosity=compact",
                        ],
                    }
                }
            }
        )
        result = await session.call_tool("get_weather", {"location": "Bern"})
        print(result)  # → compact JSON dict
```

## Design Notes

- **Session-scoped**: `ContentNegotiationMiddleware.on_initialize` runs once
  per connection; `ctx.get_state` is zero-overhead on subsequent calls.
- **No per-request overhead**: feature tags are stored in session state, not
  re-parsed on every tool call.
- **Safe defaults**: `Features` properties all have sensible defaults
  (`format="markdown"`, `verbosity="standard"`) so tools degrade gracefully
  when no negotiation occurred.
- **Security**: feature tags only influence content shape — the tool always
  computes the same underlying data and only varies the presentation.
- **Extensible**: `Features.has_tag()` and `Features.vendor_value()` support
  vendor tags (`x-mycompany-hint=value`) without changes to the middleware.
