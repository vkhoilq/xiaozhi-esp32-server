# core/providers/llm/ollama/

## Responsibility

Provides an LLM provider implementation for **Ollama** — a local LLM runner. This module bridges the voice assistant server with Ollama's OpenAI-compatible API endpoint, enabling streaming chat completions, native function/tool calling, and special handling for Qwen3 models.

## Design

### Class: `LLMProvider` (inherits `LLMProviderBase`)

**Initialization:**
- Reads `model_name`, `base_url` (default `"http://localhost:11434"`) from config.
- Appends `/v1` to the base URL if not already present, making it compatible with OpenAI client expectations.
- Creates an `OpenAI` client with a dummy `api_key="ollama"` (the OpenAI SDK requires a key, but Ollama does not validate it).
- Detects Qwen3 models (`model_name.lower().startswith("qwen3")`) for special handling.

**Key Design Decisions:**
- **OpenAI-Compatible SDK**: Uses the `openai` Python library pointing at Ollama's `/v1` endpoint, reusing OpenAI's streaming and tool-calling patterns.
- **Qwen3 `/no_think` Directive**: For Qwen3 models, a `/no_think ` prefix is prepended to the last user message to disable the model's thinking/reasoning output. This is applied in both `response()` and `response_with_functions()`.
- **Think-Tag Stripping with Buffer**: Uses a `buffer` + `is_active` state machine to handle cross-chunk `<think>` / `</think>` tags:
  - When `<think>` is detected (possibly split across chunks), text before it is queued, `is_active` becomes `False`.
  - When `</think>` is detected, text after it becomes active again.
  - Complete `<think>...</think>` blocks are fully removed.
  - Active text is yielded only when `is_active` is `True`.
- **Native Function Calling**: `response_with_functions()` passes the `tools` parameter directly to `client.chat.completions.create()`. Tool call deltas are detected via `delta.tool_calls` and yielded as `(None, tool_calls)`. Text content from the same stream is processed through the same think-tag filter.

## Flow

### Plain text flow (`response`)
1. If Qwen3 model: prepend `/no_think ` to the last user message (operates on a copy to avoid mutating the original dialogue).
2. Calls `client.chat.completions.create(model, messages, stream=True)`.
3. Iterates over chunks, extracting `delta.content`.
4. Processes content through the think-tag buffer/state machine.
5. Yields filtered text tokens.

### Function-calling flow (`response_with_functions`)
1. Same Qwen3 `/no_think` handling.
2. Calls `client.chat.completions.create(model, messages, stream=True, tools=functions)`.
3. For each chunk:
   - If `delta.tool_calls` present: yields `(None, tool_calls)`.
   - If `delta.content` present: processes through think-tag filter, yields `(filtered_content, None)`.
4. Ensures stream is closed in a `finally` block.

## Integration

### Dependencies
- `openai` (OpenAI Python SDK, used in OpenAI-compatible mode)
- `core.providers.llm.base.LLMProviderBase` (parent class)
- `config.logger` (structured logging)

### Consumed by
- The LLM provider factory/dispatch, invoked when config specifies `"type": "ollama"` (or similar alias).
- Orchestration layer calls `response()` for dialogue completion or `response_with_functions()` for intent/tool resolution.

### Notable Details
- The dummy `api_key="ollama"` is required by the OpenAI SDK but is not validated by Ollama.
- Qwen3 models require the `/no_think` prefix as a model-specific instruction; other models remain unaffected.
- The think-tag buffer handles both intra-chunk and cross-chunk `<think>` tags, making it robust to variable chunk boundaries.
