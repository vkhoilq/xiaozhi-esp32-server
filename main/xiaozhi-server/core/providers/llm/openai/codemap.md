# core/providers/llm/openai/

## Responsibility

Provides an LLM provider implementation for **OpenAI-compatible APIs** — the most flexible and widely-used adapter. This module bridges the voice assistant server with any OpenAI-format endpoint (OpenAI official, Azure, Alibaba Cloud, Moonshot, Zhipu/BigModel, Volcengine, etc.), supporting streaming chat completions, native function/tool calling, configurable generation parameters, and automatic thinking-mode suppression for specific platforms.

## Design

### Class: `LLMProvider` (inherits `LLMProviderBase`)

**Initialization:**
- Reads `model_name`, `api_key`, `base_url` (falls back to `url`), and optional `timeout`, `max_tokens`, `temperature`, `top_p`, `frequency_penalty` from config.
- **Timeout Handling**: Supports both simple numeric timeout (converted to `httpx.Timeout`) and a fine-grained dict timeout with keys `pool`, `connect`, `write`, `read`. Defaults to 300 seconds.
- **Parameter Parsing**: Uses a `param_defaults` dict with converters for each optional generation parameter. `None`/empty values result in `None` attributes (omitted from requests if `None`).
- Validates the API key via `check_model_key()`.
- Creates an `openai.OpenAI` client with the configured `api_key`, `base_url`, and `custom_timeout`.

**Key Design Decisions:**
- **Thinking Mode Suppression** (`_apply_thinking_disabled`): Detects the API domain from `base_url` and injects `extra_body` parameters to disable thinking/reasoning for specific platforms:
  - `aliyuncs.com` → `{"enable_thinking": false}`
  - `bigmodel.cn` → `{"thinking": {"type": "disabled"}}`
  - `moonshot.cn` → `{"thinking": {"type": "disabled"}}`
  - `volces.com` → `{"thinking": {"type": "disabled"}}`
- **Dialogue Normalization** (`normalize_dialogue`): Automatically fills missing `content` fields in dialogue messages with empty strings, preventing API errors from malformed messages.
- **Think-Tag Stripping**: Inline processing of `<think>` / `</think>` tags in the streaming output using an `is_active` state variable. Content within thinking tags is silently dropped.
- **Native Function Calling**: `response_with_functions()` passes `tools=functions` to the API. Yields `(content, tool_calls)` tuples from the delta. Tracks token usage via `chunk.usage` (if present) and logs prompt/completion token counts.
- **Optional Parameter Propagation**: `max_tokens`, `temperature`, `top_p`, `frequency_penalty` are sent to the API only when non-`None`. These can be overridden per-call via `kwargs`.

## Flow

### Plain text flow (`response`)
1. Normalizes the dialogue (fills missing `content` fields).
2. Builds request params dict with `model`, `messages`, `stream=True`, and optional generation params.
3. Applies thinking-mode suppression via `_apply_thinking_disabled()`.
4. Calls `client.chat.completions.create(**request_params)`.
5. Iterates over chunks:
   - Extracts `delta.content` from `chunk.choices[0].delta`.
   - Strips `<think>` / `</think>` segments using `is_active` state.
   - Yields clean text tokens.

### Function-calling flow (`response_with_functions`)
1. Same dialogue normalization and param building, with `tools=functions` added.
2. Iterates over chunks:
   - Extracts `delta.content` and `delta.tool_calls`.
   - Yields `(content, tool_calls)` — either or both can be non-None.
   - On `chunk.usage` (final chunk with token counts): logs token consumption.
3. Ensures stream is closed in a `finally` block.

## Integration

### Dependencies
- `openai` (official OpenAI Python SDK)
- `httpx` (used internally by the OpenAI SDK for fine-grained timeout configuration)
- `core.providers.llm.base.LLMProviderBase` (parent class)
- `core.utils.util.check_model_key()` (API key validation)
- `config.logger` (structured logging)

### Consumed by
- The LLM provider factory/dispatch, invoked when config specifies `"type": "openai"` (or similar alias, or any OpenAI-compatible service).
- Orchestration layer calls `response()` for dialogue completion or `response_with_functions()` for intent/tool resolution.
- This is the most commonly used provider, serving as the default adapter for any OpenAI-compatible backend.

### Notable Details
- The thinking-mode suppression is a pragmatic workaround for Chinese AI platforms that emit verbose reasoning tokens by default.
- The `httpx.Timeout` granularity (pool/connect/write/read) allows fine-tuning for different network conditions.
- Token usage logging in function-calling mode provides visibility into API consumption costs.
