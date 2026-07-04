# Provider-Agnostic AI Model Abstraction

```
Version:        1.1.0
Last Updated:   2026-06-04
Applicability:  Projects integrating multiple LLM providers (Claude, GPT, Gemini) or needing provider portability
Dependencies:   anthropic SDK, openai SDK, google-generativeai SDK (install only providers you use)
```

---

## When to Abstract

**Do abstract** when:
- You need to support multiple providers (user choice, A/B testing, fallback).
- You want to swap providers without rewriting application code.
- You're building a platform where tenants bring their own API keys.

**Don't abstract** when:
- You're committed to a single provider for the foreseeable future.
- You rely heavily on provider-specific features (Claude tool use, OpenAI assistants, Gemini grounding).
- Premature abstraction will slow initial development.

**Pragmatic middle ground:** Start with a thin interface. One concrete implementation. Add adapters only when you actually need a second provider.

---

## Interface Definition

### Core Types

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from enum import Enum
from typing import AsyncIterator

class Role(Enum):
    SYSTEM = "system"
    USER = "user"
    ASSISTANT = "assistant"
    TOOL_RESULT = "tool_result"

@dataclass
class Message:
    role: Role
    content: str
    tool_calls: list["ToolCall"] = field(default_factory=list)
    tool_results: list["ToolResult"] = field(default_factory=list)

@dataclass
class ToolDefinition:
    name: str
    description: str
    parameters: dict  # JSON Schema

@dataclass
class ToolCall:
    id: str
    name: str
    arguments: dict

@dataclass
class ToolResult:
    tool_call_id: str
    content: str
    is_error: bool = False

@dataclass
class ModelResponse:
    content: str
    tool_calls: list[ToolCall]
    finish_reason: str          # "stop", "tool_use", "max_tokens"
    usage: "TokenUsage"
    raw_response: object        # Provider-specific response for escape hatch

@dataclass
class TokenUsage:
    input_tokens: int
    output_tokens: int
    total_tokens: int

@dataclass
class StreamChunk:
    text: str | None = None
    tool_call: ToolCall | None = None
    is_final: bool = False
    usage: TokenUsage | None = None  # Only on final chunk
```

### Provider Interface

```python
class LLMProvider(ABC):
    @abstractmethod
    async def complete(
        self,
        messages: list[Message],
        model: str,
        max_tokens: int = 4096,
        temperature: float = 0.0,
        tools: list[ToolDefinition] | None = None,
        system: str | None = None,
    ) -> ModelResponse:
        """Send a completion request and return the full response."""
        ...

    @abstractmethod
    async def stream(
        self,
        messages: list[Message],
        model: str,
        max_tokens: int = 4096,
        temperature: float = 0.0,
        tools: list[ToolDefinition] | None = None,
        system: str | None = None,
    ) -> AsyncIterator[StreamChunk]:
        """Stream a completion response."""
        ...

    @abstractmethod
    def count_tokens(self, text: str, model: str) -> int:
        """Estimate token count for the given text."""
        ...
```

---

## Adapter Implementations

### Claude Adapter

```python
import anthropic

class ClaudeProvider(LLMProvider):
    def __init__(self, api_key: str | None = None):
        self.client = anthropic.AsyncAnthropic(api_key=api_key)

    async def complete(self, messages, model, max_tokens=4096,
                       temperature=0.0, tools=None, system=None):
        api_messages = self._convert_messages(messages)
        api_tools = self._convert_tools(tools) if tools else None

        kwargs = {
            "model": model,
            "max_tokens": max_tokens,
            "temperature": temperature,
            "messages": api_messages,
        }
        if system:
            kwargs["system"] = system
        if api_tools:
            kwargs["tools"] = api_tools

        response = await self.client.messages.create(**kwargs)
        return self._parse_response(response)

    async def stream(self, messages, model, max_tokens=4096,
                     temperature=0.0, tools=None, system=None):
        api_messages = self._convert_messages(messages)
        kwargs = {
            "model": model,
            "max_tokens": max_tokens,
            "temperature": temperature,
            "messages": api_messages,
        }
        if system:
            kwargs["system"] = system

        async with self.client.messages.stream(**kwargs) as stream:
            async for text in stream.text_stream:
                yield StreamChunk(text=text)
            final = await stream.get_final_message()
            yield StreamChunk(
                is_final=True,
                usage=TokenUsage(
                    input_tokens=final.usage.input_tokens,
                    output_tokens=final.usage.output_tokens,
                    total_tokens=final.usage.input_tokens + final.usage.output_tokens,
                ),
            )

    def count_tokens(self, text, model):
        # Use anthropic's token counter or approximate: ~4 chars per token
        return len(text) // 4

    def _convert_messages(self, messages: list[Message]) -> list[dict]:
        result = []
        for msg in messages:
            if msg.role == Role.SYSTEM:
                continue  # Handled separately in Claude API
            result.append({
                "role": "user" if msg.role == Role.USER else "assistant",
                "content": msg.content,
            })
        return result

    def _convert_tools(self, tools: list[ToolDefinition]) -> list[dict]:
        return [
            {
                "name": t.name,
                "description": t.description,
                "input_schema": t.parameters,
            }
            for t in tools
        ]

    def _parse_response(self, response) -> ModelResponse:
        text_parts = []
        tool_calls = []
        for block in response.content:
            if block.type == "text":
                text_parts.append(block.text)
            elif block.type == "tool_use":
                tool_calls.append(ToolCall(
                    id=block.id, name=block.name, arguments=block.input,
                ))
        return ModelResponse(
            content="".join(text_parts),
            tool_calls=tool_calls,
            finish_reason="tool_use" if response.stop_reason == "tool_use" else "stop",
            usage=TokenUsage(
                input_tokens=response.usage.input_tokens,
                output_tokens=response.usage.output_tokens,
                total_tokens=response.usage.input_tokens + response.usage.output_tokens,
            ),
            raw_response=response,
        )
```

### OpenAI Adapter

```python
from openai import AsyncOpenAI
import json

class OpenAIProvider(LLMProvider):
    def __init__(self, api_key: str | None = None):
        self.client = AsyncOpenAI(api_key=api_key)

    async def complete(self, messages, model, max_tokens=4096,
                       temperature=0.0, tools=None, system=None):
        api_messages = self._convert_messages(messages, system)
        api_tools = self._convert_tools(tools) if tools else None

        kwargs = {
            "model": model,
            "max_tokens": max_tokens,
            "temperature": temperature,
            "messages": api_messages,
        }
        if api_tools:
            kwargs["tools"] = api_tools

        response = await self.client.chat.completions.create(**kwargs)
        return self._parse_response(response)

    def _convert_messages(self, messages, system=None):
        result = []
        if system:
            result.append({"role": "system", "content": system})
        for msg in messages:
            if msg.role == Role.SYSTEM:
                result.append({"role": "system", "content": msg.content})
            elif msg.role == Role.USER:
                result.append({"role": "user", "content": msg.content})
            elif msg.role == Role.ASSISTANT:
                result.append({"role": "assistant", "content": msg.content})
        return result

    def _convert_tools(self, tools):
        return [
            {
                "type": "function",
                "function": {
                    "name": t.name,
                    "description": t.description,
                    "parameters": t.parameters,
                },
            }
            for t in tools
        ]

    def _parse_response(self, response) -> ModelResponse:
        choice = response.choices[0]
        tool_calls = []
        if choice.message.tool_calls:
            for tc in choice.message.tool_calls:
                tool_calls.append(ToolCall(
                    id=tc.id,
                    name=tc.function.name,
                    arguments=json.loads(tc.function.arguments),
                ))
        return ModelResponse(
            content=choice.message.content or "",
            tool_calls=tool_calls,
            finish_reason="tool_use" if choice.finish_reason == "tool_calls" else "stop",
            usage=TokenUsage(
                input_tokens=response.usage.prompt_tokens,
                output_tokens=response.usage.completion_tokens,
                total_tokens=response.usage.total_tokens,
            ),
            raw_response=response,
        )

    # stream() and count_tokens() follow same pattern...
```

---

## Adapter Pattern Architecture

```
Application Code
       |
       v
  LLMProvider (interface)
       |
  +-----------+-----------+-----------+
  |           |           |           |
ClaudeProvider  OpenAIProvider  GeminiProvider
  |           |           |
Anthropic SDK  OpenAI SDK  Google SDK
```

### Provider Registry

```python
class ProviderRegistry:
    def __init__(self):
        self._providers: dict[str, LLMProvider] = {}
        self._model_map: dict[str, str] = {}  # model alias -> provider name

    def register(self, name: str, provider: LLMProvider, models: list[str]):
        self._providers[name] = provider
        for model in models:
            self._model_map[model] = name

    def get_provider(self, model: str) -> LLMProvider:
        provider_name = self._model_map.get(model)
        if not provider_name:
            raise ValueError(f"No provider registered for model: {model}")
        return self._providers[provider_name]

# Usage
registry = ProviderRegistry()
registry.register("anthropic", ClaudeProvider(), [
    "claude-opus-4-0520", "claude-sonnet-4-20250514",
])
registry.register("openai", OpenAIProvider(), [
    "gpt-4o", "gpt-4o-mini",
])

# Application code just uses model names
provider = registry.get_provider("claude-sonnet-4-20250514")
response = await provider.complete(messages, model="claude-sonnet-4-20250514")
```

---

## Capability Negotiation

Not all providers support all features. Model capabilities should be queryable.

```python
@dataclass
class ModelCapabilities:
    max_context_tokens: int
    max_output_tokens: int
    supports_tools: bool
    supports_vision: bool
    supports_streaming: bool
    supports_system_prompt: bool  # Gemini handles this differently
    supports_json_mode: bool
    supports_extended_thinking: bool

CAPABILITY_MAP: dict[str, ModelCapabilities] = {
    "claude-opus-4-0520": ModelCapabilities(
        max_context_tokens=200_000, max_output_tokens=32_000,
        supports_tools=True, supports_vision=True, supports_streaming=True,
        supports_system_prompt=True, supports_json_mode=False,
        supports_extended_thinking=True,
    ),
    "claude-sonnet-4-20250514": ModelCapabilities(
        max_context_tokens=200_000, max_output_tokens=16_000,
        supports_tools=True, supports_vision=True, supports_streaming=True,
        supports_system_prompt=True, supports_json_mode=False,
        supports_extended_thinking=True,
    ),
    "gpt-4o": ModelCapabilities(
        max_context_tokens=128_000, max_output_tokens=16_384,
        supports_tools=True, supports_vision=True, supports_streaming=True,
        supports_system_prompt=True, supports_json_mode=True,
        supports_extended_thinking=False,
    ),
    "gemini-2.0-flash": ModelCapabilities(
        max_context_tokens=1_000_000, max_output_tokens=8_192,
        supports_tools=True, supports_vision=True, supports_streaming=True,
        supports_system_prompt=True, supports_json_mode=True,
        supports_extended_thinking=True,
    ),
}

def get_capabilities(model: str) -> ModelCapabilities:
    if model not in CAPABILITY_MAP:
        raise ValueError(f"Unknown model: {model}")
    return CAPABILITY_MAP[model]
```

### Graceful Degradation

```python
async def complete_with_negotiation(
    provider: LLMProvider,
    model: str,
    messages: list[Message],
    system: str | None = None,
    tools: list[ToolDefinition] | None = None,
) -> ModelResponse:
    caps = get_capabilities(model)

    # Degrade gracefully for unsupported features
    if not caps.supports_system_prompt and system:
        # Prepend system prompt as a user message
        messages = [Message(role=Role.USER, content=f"[System]: {system}")] + messages
        system = None

    if not caps.supports_tools and tools:
        # Inject tool descriptions into system prompt
        tool_desc = "\n".join(
            f"- {t.name}: {t.description}" for t in tools
        )
        system = (system or "") + f"\n\nAvailable tools:\n{tool_desc}\nTo use a tool, respond with JSON: {{\"tool\": \"name\", \"args\": {{...}}}}"
        tools = None

    return await provider.complete(messages, model, system=system, tools=tools)
```

---

## Fallback Strategies

### Cascading Fallback

```python
class FallbackProvider(LLMProvider):
    def __init__(self, providers: list[tuple[str, LLMProvider]]):
        """providers: list of (model, provider) pairs in priority order."""
        self.providers = providers

    async def complete(self, messages, model=None, **kwargs):
        last_error = None
        for fallback_model, provider in self.providers:
            try:
                return await provider.complete(
                    messages, model=fallback_model, **kwargs
                )
            except Exception as e:
                last_error = e
                logger.warning(f"Provider {fallback_model} failed: {e}. Trying next.")
                continue
        raise last_error

# Priority: Claude Sonnet -> GPT-4o -> Gemini Flash
fallback = FallbackProvider([
    ("claude-sonnet-4-20250514", ClaudeProvider()),
    ("gpt-4o", OpenAIProvider()),
    ("gemini-2.0-flash", GeminiProvider()),
])
```

### Latency-Based Routing

```python
import asyncio
import time

class LatencyRouter(LLMProvider):
    def __init__(self, providers: dict[str, tuple[str, LLMProvider]]):
        self.providers = providers
        self.latencies: dict[str, float] = {k: 0.0 for k in providers}

    async def complete(self, messages, model=None, **kwargs):
        # Pick the provider with lowest recent latency
        sorted_providers = sorted(self.latencies.items(), key=lambda x: x[1])

        for name, _ in sorted_providers:
            target_model, provider = self.providers[name]
            try:
                start = time.monotonic()
                result = await provider.complete(messages, model=target_model, **kwargs)
                elapsed = time.monotonic() - start
                # Exponential moving average
                self.latencies[name] = 0.7 * self.latencies[name] + 0.3 * elapsed
                return result
            except Exception:
                self.latencies[name] = float("inf")  # Penalize failures
                continue
        raise RuntimeError("All providers failed")
```

---

## Cost Normalization

Normalize costs across providers to enable cost-aware routing.

```python
# Cost per 1M tokens (input, output)
COST_TABLE: dict[str, tuple[float, float]] = {
    # Anthropic
    "claude-opus-4-0520":           (15.0, 75.0),
    "claude-sonnet-4-20250514":     (3.0, 15.0),
    "claude-3-5-haiku-20241022":    (0.80, 4.0),
    # OpenAI
    "gpt-4o":                       (2.50, 10.0),
    "gpt-4o-mini":                  (0.15, 0.60),
    # Google
    "gemini-2.0-flash":             (0.10, 0.40),
    "gemini-2.0-pro":               (1.25, 10.0),
}

def compute_cost(model: str, usage: TokenUsage) -> float:
    input_rate, output_rate = COST_TABLE[model]
    return (usage.input_tokens * input_rate + usage.output_tokens * output_rate) / 1_000_000

class CostTracker:
    def __init__(self):
        self.total_cost = 0.0
        self.costs_by_model: dict[str, float] = {}

    def record(self, model: str, usage: TokenUsage):
        cost = compute_cost(model, usage)
        self.total_cost += cost
        self.costs_by_model[model] = self.costs_by_model.get(model, 0.0) + cost
        return cost
```

### Cost-Aware Model Selection

```python
def select_model(task_complexity: str, max_cost_per_request: float) -> str:
    """Select the best model within budget."""
    tiers = {
        "simple":  ["claude-3-5-haiku-20241022", "gpt-4o-mini", "gemini-2.0-flash"],
        "moderate": ["claude-sonnet-4-20250514", "gpt-4o", "gemini-2.0-pro"],
        "complex": ["claude-opus-4-0520", "gpt-4o", "claude-sonnet-4-20250514"],
    }

    candidates = tiers.get(task_complexity, tiers["moderate"])

    for model in candidates:
        # Estimate cost for typical request (1K input, 1K output)
        input_rate, output_rate = COST_TABLE[model]
        estimated = (1000 * input_rate + 1000 * output_rate) / 1_000_000
        if estimated <= max_cost_per_request:
            return model

    return candidates[-1]  # Fallback to cheapest in tier
```

---

## DB-Backed Model Registry (with Code Fallback)

A hardcoded `COST_TABLE` / model list means every model add, retirement, price
change, or default switch is a code edit plus a redeploy. Once that cadence
hurts (or you need per-tenant model access / margin pricing later), move the
list into a database registry that both the runtime and the UI read from, while
keeping the hardcoded table as a SAFE FALLBACK so a registry problem can never
break inference or cost tracking.

Shape:

1. A `models` table: model_string (unique), label, input_cost / output_cost (in
   whatever unit your cost math already uses, so the math is unchanged),
   enabled, is_default, sort_order, plus retirement metadata. Enforce a single
   default with a partial unique index (`UNIQUE (is_default) WHERE is_default`);
   the set-default write clears the prior default first so the switch never
   trips the index. Seed the table FROM the current hardcoded values, verbatim
   and idempotently, so the migration changes nothing.
2. Keep `compute_cost` SYNCHRONOUS and fast (it runs on every call). Do NOT
   read the DB per call. Load the registry into an in-memory overlay at startup
   and refresh it after each registry edit; `compute_cost` consults the overlay
   first, then the hardcoded table, then 0.0 as the last resort. Resolve the
   default the same way (overlay default, then the constant).
3. Two read paths with different privilege: operator/admin endpoints expose the
   full rows (including costs) for management; the public model PICKER reads a
   cost-stripped endpoint (model_string + label + is_default for enabled models
   only). Keep costs out of any client that does not need them, and prefer
   serving the public list through an endpoint over a second cost-stripped RLS
   policy.
4. Every read falls back to the constants: the UI picker falls back to the
   hardcoded list if the fetch fails (never leave the user unable to pick), and
   the runtime falls back to the constant cost table. Editing the constants now
   only changes the fallback.

A registry lets an operator APPLY a known model with no code edit; it cannot
auto-detect a provider's new releases (there is no reliable signal), so the
operator enters a new model when they learn of it. Say so in the UI.

---

## Key Design Principles

1. **Keep the interface minimal.** `complete()`, `stream()`, and `count_tokens()` cover 95% of use cases. Don't abstract features you don't use.

2. **Preserve the escape hatch.** Always include `raw_response` so callers can access provider-specific features without breaking the abstraction.

3. **Normalize at the boundary.** Convert provider-specific types to your types immediately in the adapter. Application code never touches SDK objects.

4. **Test with real APIs.** Mock-based tests miss format differences between providers. Run integration tests against each provider periodically.

5. **Version your model references.** Use dated model IDs (`claude-sonnet-4-20250514`) not aliases (`claude-sonnet`) in production to avoid surprise behavior changes.

6. **Don't over-normalize prompts.** Each model has different strengths. A prompt optimized for Claude may underperform on GPT. Allow per-provider prompt overrides when quality matters.
