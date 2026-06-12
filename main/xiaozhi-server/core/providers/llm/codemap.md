# core/providers/llm/

## Responsibility

Provides a **plug-in abstraction layer** for Large Language Model (LLM) backends. Defines the `LLMProviderBase` interface that all provider implementations must satisfy, enabling the voice assistant server to delegate dialogue completion, function calling, and intent recognition to any supported LLM service interchangeably.

## Design

### Abstract Base Class (`base.py`)
- **`LLMProviderBase`** (ABC) defines three key methods:
  - `response(session_id, dialogue)` → `Generator[str]` - Abstract, must be implemented by each provider. Yields text tokens via streaming.
  - `response_no_stream(system_prompt, user_prompt)` → `str` - Convenience wrapper that constructs a dialogue from system/user prompts and collects the full stream into a single string.
  - `response_with_functions(session_id, dialogue, functions=None)` → `Generator[Tuple[str, Any]]` - Default implementation falls back to plain `response()`, yielding `(token, None)`. Providers that support tool/function calling override this to yield `(token, tool_calls)` tuples.

### System Prompt Factory (`system_prompt.py`)
- `get_system_prompt_for_function(functions: str)` returns a raw prompt string containing:
  - Tool-use formatting guidelines (JSON-style `<tool_call>` tags with `name` and `arguments`)
  - The available function definitions injected into the prompt
  - Step-by-step instructions for tool use
- This prompt is used by providers that lack native function-calling support (Coze, Dify) to simulate function calling via prompt engineering.

### Provider Registration Pattern
- Each subdirectory contains a single `LLMProvider` class that inherits from `LLMProviderBase`.
- Providers are instantiated via a factory/dispatch pattern in the caller (see **Integration**), selected by config key.

## Flow

1. **Caller** (outside this module) instantiates the appropriate `LLMProvider(config)` using the configuration's provider type and credentials.
2. On a user utterance, the caller assembles a `dialogue` list (OpenAI-style: `[{"role": "system"|"user"|"assistant", "content": "..."}]`).
3. **Streaming path** (`response`): The caller iterates over the generator to receive text tokens in real-time for TTS playback.
4. **Function-calling path** (`response_with_functions`):
   - If the provider supports native tool calling (OpenAI, Ollama, Xinference, Gemini), it delegates to the SDK's `tools` parameter and yields structured tool_calls.
   - If the provider does not (Coze, Dify), the function definitions are serialized into the system prompt to simulate tool use.
5. The caller consumes the `(text_token, tool_calls)` tuples, handling function execution or text delivery accordingly.

## Integration

### Dependencies (external)
- `config.logger` - Structured logging (structlog-based)
- `core.utils.util.check_model_key()` - Validates API key presence on init
- Each provider has its own SDK dependencies (openai, dashscope, cozepy, google-generativeai, httpx, requests)

### Consumed by
- **`core/processor.py`** or similar orchestration layer — selects the provider class based on config and calls `response()` / `response_with_functions()` for each user turn.
- **Intent recognition pipeline** — uses `response_with_functions()` to detect when the assistant should execute a tool/action rather than speak.

### Provided implementations (subdirectories)
| Provider | Key SDK/Transport | Function Calling | Streaming |
|---|---|---|---|
| `AliBL/` | DashScope SDK | No (prompt fallback) | Yes |
| `coze/` | CozePy SDK | No (prompt engineering) | Yes |
| `dify/` | HTTP (requests, SSE) | No (prompt engineering) | Yes |
| `fastgpt/` | HTTP (requests, SSE) | No (unsupported) | Yes |
| `gemini/` | google-generativeai SDK | Yes (native FunctionDeclaration) | Yes |
| `homeassistant/` | HTTP (requests, REST) | No (unsupported) | No |
| `ollama/` | OpenAI-compatible SDK | Yes (tools param) | Yes |
| `openai/` | OpenAI SDK + httpx | Yes (tools param) | Yes |
| `xinference/` | OpenAI-compatible SDK | Yes (tools param) | Yes |
