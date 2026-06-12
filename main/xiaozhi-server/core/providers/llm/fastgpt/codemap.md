# core/providers/llm/fastgpt/

## Responsibility

Provides an LLM provider implementation for **FastGPT** — an open-source knowledge-based Q&A platform with an OpenAI-compatible API. This module bridges the voice assistant server with FastGPT's chat completion endpoint, streaming text responses over HTTP SSE.

## Design

### Class: `LLMProvider` (inherits `LLMProviderBase`)

**Initialization:**
- Reads `api_key`, `base_url`, `detail` (boolean), and `variables` (dict) from config.
- Validates the API key via `check_model_key()`.

**Key Design Decisions:**
- **OpenAI-Compatible Endpoint**: Calls `{base_url}/chat/completions` with an OpenAI-style request body including `stream`, `chatId`, `detail`, `variables`, and `messages`.
- **Single-User-Message Extraction**: Only the last `role: "user"` message is extracted from the dialogue and sent as a single-element messages array. No system prompt or history is forwarded — context is managed server-side by FastGPT.
- **SSE Streaming**: Uses `requests.post(stream=True)` and `r.iter_lines()`. Parses `data:` lines as JSON, extracts `choices[0].delta.content` tokens.
- **Think-Tag Filtering**: Tokens containing `<think>` or `</think>` tags are silently skipped (these are reasoning tokens from models like DeepSeek).
- **Termination Detection**: Stops iterating when a `data: [DONE]` line is encountered.
- **No Function Calling**: `response_with_functions()` logs an error that FastGPT does not support function calling and returns nothing.

## Flow

1. `response(session_id, dialogue)` extracts the last user message from the dialogue.
2. POSTs to `{base_url}/chat/completions` with `Authorization: Bearer {api_key}`, body containing `stream: True`, `chatId: session_id`, `detail`, `variables`, and the user message.
3. Iterates over SSE lines:
   - Skips non-`data:` lines.
   - Breaks on `[DONE]`.
   - Parses JSON, extracts `delta.content` from the first choice.
   - Skips tokens containing `</?think>`.
   - Yields clean content tokens.

## Integration

### Dependencies
- `requests` (HTTP client for SSE streaming)
- `core.providers.llm.base.LLMProviderBase` (parent class)
- `config.logger` (structured logging)

### Consumed by
- The LLM provider factory/dispatch, invoked when config specifies `"type": "fastgpt"` (or similar alias).
- Orchestration layer calls `response()` for dialogue completion.

### Limitations
- No native function calling — `response_with_functions()` is explicitly unsupported and logs an error if called.
- No multi-turn conversation history sent to FastGPT (only the last user message is forwarded). The server is expected to manage context internally.
- No conversation ID tracking or session persistence — each request is treated independently.
