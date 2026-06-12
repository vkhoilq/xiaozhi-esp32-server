# core/providers/llm/xinference/

## Responsibility

Provides an LLM provider implementation for **Xorbits Inference (Xinference)** — an open-source platform for serving large language models. This module bridges the voice assistant server with Xinference's OpenAI-compatible API endpoint, enabling streaming chat completions and native function/tool calling for locally or self-hosted models.

## Design

### Class: `LLMProvider` (inherits `LLMProviderBase`)

**Initialization:**
- Reads `model_name`, `base_url` (default `"http://localhost:9997"`) from config.
- Appends `/v1` to the base URL if not already present, making it compatible with OpenAI client expectations.
- Creates an `OpenAI` client with a dummy `api_key="xinference"` (the OpenAI SDK requires a key, but Xinference has its own auth mechanism and does not validate this placeholder).
- Logs initialization success/failure.

**Key Design Decisions:**
- **OpenAI-Compatible SDK**: Uses the `openai` Python library pointing at Xinference's `/v1` endpoint, reusing OpenAI's streaming and tool-calling patterns.
- **Think-Tag Stripping**: Inline filtering of `<think>` / `</think>` tags using an `is_active` state variable. Content within thinking tags is silently dropped. This is simpler than Ollama's buffer-based approach — operates per-chunk only.
- **Native Function Calling**: `response_with_functions()` passes `tools=functions` directly to `client.chat.completions.create()`. For each chunk, extracts both `delta.content` and `delta.tool_calls`. Yields `(content, tool_calls)` when content is present, or `(None, tool_calls)` for tool call deltas.
- **Stream Cleanup**: The function-calling path closes the stream in a `finally` block to ensure proper resource cleanup.

## Flow

### Plain text flow (`response`)
1. Calls `client.chat.completions.create(model, messages=dialogue, stream=True)`.
2. Iterates over chunks:
   - Extracts `delta.content`.
   - Strips `<think>` / `</think>` segments using `is_active` toggle.
   - Yields clean text tokens.
3. Errors per-chunk are caught and logged without crashing the stream.

### Function-calling flow (`response_with_functions`)
1. Calls `client.chat.completions.create(model, messages=dialogue, stream=True, tools=functions)`.
2. For each chunk:
   - Extracts `delta.content` and `delta.tool_calls`.
   - Yields `(content, tool_calls)` or `(None, tool_calls)` as appropriate.
3. Closes the stream in `finally`.

## Integration

### Dependencies
- `openai` (OpenAI Python SDK, used in OpenAI-compatible mode)
- `core.providers.llm.base.LLMProviderBase` (parent class)
- `config.logger` (structured logging)

### Consumed by
- The LLM provider factory/dispatch, invoked when config specifies `"type": "xinference"` (or similar alias).
- Orchestration layer calls `response()` for dialogue completion or `response_with_functions()` for intent/tool resolution.

### Notable Details
- Very similar to the Ollama provider in structure — both use `openai.OpenAI` pointing at a local `/v1` endpoint with a dummy API key.
- The default port is `9997` (Xinference's default), whereas Ollama defaults to `11434`.
- Think-tag stripping is chunk-local (no buffer for cross-chunk tags), unlike Ollama's more robust buffer-based approach. This means split `<think>` tags across chunk boundaries may not be handled correctly.
- The placeholder API key `"xinference"` is not validated; Xinference deployments may require their own authentication via the `api_key` config if configured server-side.
