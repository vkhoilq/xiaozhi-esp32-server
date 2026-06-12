# core/providers/llm/homeassistant/

## Responsibility

Provides an LLM provider implementation for **Home Assistant** — an open-source home automation platform. This module bridges the voice assistant server with Home Assistant's built-in conversation API (`/api/conversation/process`), delegating natural language understanding and response generation to Home Assistant's configured voice/assistant pipeline (e.g., Assist, configured LLM or local intent matching).

## Design

### Class: `LLMProvider` (inherits `LLMProviderBase`)

**Initialization:**
- Reads `agent_id`, `api_key`, `base_url` (falls back to `url`) from config.
- Constructs the API URL as `{base_url}/api/conversation/process`.

**Key Design Decisions:**
- **Non-Streaming**: Unlike most other providers, Home Assistant returns a complete JSON response (not streaming). The entire response is received and parsed before any output is yielded.
- **Stateless Per-Request**: Uses `session_id` directly as the `conversation_id` parameter for Home Assistant's built-in conversation history management. No local conversation map is maintained.
- **User Message Extraction**: Iterates over the dialogue list in reverse to find the last `role: "user"` message. This is the only content sent — Home Assistant handles context internally via `conversation_id`.
- **Response Parsing**: Extracts `response.speech.plain.speech` from the nested JSON response structure. If `speech` is empty/missing, logs a warning and yields nothing.
- **No Function Calling**: `response_with_functions()` logs an error that Home Assistant does not support function calling and returns nothing.

## Flow

1. `response(session_id, dialogue)` extracts the last user message text.
2. Constructs a JSON payload with `text`, `agent_id`, and `conversation_id` (= `session_id`).
3. POSTs to `{base_url}/api/conversation/process` with `Authorization: Bearer {api_key}`.
4. Calls `response.raise_for_status()` to surface HTTP errors.
5. Parses the JSON response and navigates the nested `response.speech.plain.speech` path.
6. If speech content exists, yields it as a single string (non-streaming).

## Integration

### Dependencies
- `requests` (HTTP client)
- `core.providers.llm.base.LLMProviderBase` (parent class)
- `config.logger` (structured logging)

### Consumed by
- The LLM provider factory/dispatch, invoked when config specifies `"type": "homeassistant"` (or similar alias).
- Orchestration layer calls `response()` for dialogue completion.

### Limitations
- **Non-streaming**: Not suitable for real-time TTS streaming; the entire response is buffered before output begins.
- **No function calling**: `response_with_functions()` is unsupported and logs an error if called.
- **HA-Specific Format**: The response parsing assumes the exact Home Assistant API response structure (`response.speech.plain.speech`), which may change across HA versions.
