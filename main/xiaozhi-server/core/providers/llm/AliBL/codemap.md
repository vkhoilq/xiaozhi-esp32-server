# core/providers/llm/AliBL/

## Responsibility

Provides an LLM provider implementation for **Alibaba Cloud Bailian (百练)** — an enterprise LLM application platform. This module bridges the voice assistant server with Alibaba's DashScope API to generate streaming text completions from a configured Bailian application.

## Design

### Class: `LLMProvider` (inherits `LLMProviderBase`)

**Initialization:**
- Reads `api_key`, `app_id`, optional `base_url`, `is_no_prompt` flag, `ali_memory_id`, and `streaming_chunk_size` from config.
- Validates the API key via `check_model_key()`.

**Key Design Decisions:**
- **Memory Support**: When `ali_memory_id` is set, the provider passes `memory_id` and `prompt` (the last user message) to enable Bailian's memory feature for multi-turn context persistence.
- **System Prompt Control**: The `is_no_prompt` flag, when truthy, strips the first dialogue message (presumed to be the system prompt) before sending to Bailian, because Bailian applications manage their own prompt configuration server-side.
- **Streaming via SDK**: Uses DashScope's `Application.call()` with `stream=True` to get an iterable response, processing incremental deltas by comparing `current_text` vs `last_text`.
- **Fallback to Non-Streaming**: If the SDK response is not iterable (`TypeError` caught), falls back to chunking the full response text at `streaming_chunk_size` intervals.
- **Custom Base URL**: If `base_url` contains `/api/`, it overrides `dashscope.base_http_api_url` for compatibility-mode deployments.

## Flow

1. `response(session_id, dialogue)` is called with the full dialogue history.
2. If `is_no_prompt` is set, `dialogue.pop(0)` removes the system prompt.
3. Builds `call_params` dict with `api_key`, `app_id`, `session_id`, `messages=dialogue`, `stream=True`.
4. If `ali_memory_id` is configured, extracts the last user message as `prompt` and adds `memory_id` and `prompt` to params.
5. Calls `Application.call(**call_params)`:
   - **Streaming path**: Iterates over `responses`, computes delta between consecutive `output.text` values, yields each delta chunk.
   - **Non-streaming fallback**: On `TypeError`, treats `responses` as a single response, chunks `output.text` by `streaming_chunk_size`, yields each chunk.
6. `response_with_functions()` logs a warning that Bailian does not support native function calling, then falls back to `response()`, yielding `(token, None)` for compatibility.

## Integration

### Dependencies
- `dashscope` (Alibaba Cloud DashScope SDK)
- `core.providers.llm.base.LLMProviderBase` (parent class)
- `config.logger` (structured logging)

### Consumed by
- The LLM provider factory/dispatch, invoked when config specifies `"type": "AliBL"` (or similar alias).
- The orchestration layer calls `response()` for dialogue completion.

### Limitations
- No native function/tool calling; `response_with_functions()` is a passthrough to plain text generation.
- Stream delta computation assumes sequential token ordering; rare reordering may drop or repeat content.
