# core/providers/llm/dify/

## Responsibility

Provides an LLM provider implementation for **Dify** — an open-source LLM application development platform. This module bridges the voice assistant server with Dify's conversation API over HTTP, supporting three operational modes (chat-messages, workflows/run, completion-messages) with streaming SSE responses and per-session conversation tracking.

## Design

### Class: `LLMProvider` (inherits `LLMProviderBase`)

**Initialization:**
- Reads `api_key`, `mode`, `base_url` from config. Defaults mode to `"chat-messages"` and base URL to `"https://api.dify.ai/v1"`.
- Validates the API key via `check_model_key()`.
- Maintains `session_conversation_map` — maps `session_id` → Dify `conversation_id`, enabling multi-turn continuity.

**Key Design Decisions:**
- **Three Operation Modes**:
  1. `chat-messages` — Standard chat completion with `query` and optional `conversation_id`. Yields `answer` fields from SSE events, filtering out `message_replace` events.
  2. `workflows/run` — Workflow execution with `inputs.query`. Yields only the final `outputs.answer` from the `workflow_finished` event.
  3. `completion-messages` — Completion mode, yields `answer` fields (same SSE processing as chat-messages but without conversation tracking).
- **SSE Streaming via `requests`**: Uses `requests.post(stream=True)` and `r.iter_lines()` to process Server-Sent Events line by line. Each `data:` line is parsed as JSON.
- **Conversation ID Persistence**: On first response in `chat-messages` mode, extracts `conversation_id` from the event data and stores it in the session map for re-use across turns.
- **Function Calling via Prompt Injection**: Same pattern as Coze — `response_with_functions()` serializes functions into JSON, injects them via `get_system_prompt_for_function()` as a prefix to the first user message. Tool results (role="tool") are folded into the preceding user message.

## Flow

### Plain text flow (`response`)
1. Extract last user message from `dialogue`.
2. Look up `conversation_id` from `session_conversation_map`.
3. POST to `{base_url}/{mode}` with `Authorization: Bearer {api_key}` header, streaming enabled.
4. Iterate over lines:
   - **chat-messages**: Parse `data:` lines, capture `conversation_id` on first event, yield `answer`, skip `message_replace` events.
   - **workflows/run**: Parse `data:` lines, on `workflow_finished` event with `succeeded` status, yield `data.outputs.answer`.
   - **completion-messages**: Parse `data:` lines, skip `message_replace`, yield `answer`.

### Function-calling flow (`response_with_functions`)
1. If dialogue has exactly 2 messages and functions exist: inject tool-use prompt into last user message.
2. If last message is `role="tool"`: fold tool result into preceding user message.
3. Falls back to `response()`, yielding `(token, None)` — Dify function calling is simulated, not native.

## Integration

### Dependencies
- `requests` (HTTP client for SSE streaming)
- `core.providers.llm.base.LLMProviderBase` (parent class)
- `core.providers.llm.system_prompt.get_system_prompt_for_function()` (prompt injection)
- `config.logger` (structured logging)

### Consumed by
- The LLM provider factory/dispatch, invoked when config specifies `"type": "dify"` (or similar alias).
- Orchestration layer calls `response()` for dialogue completion or `response_with_functions()` for intent resolution.

### Notable Details
- SSE parsing is naive (splits on `data: ` prefix). Dify uses standard SSE format so this works reliably.
- `workflows/run` mode only yields a single final answer, making it unsuitable for real-time incremental TTS use cases.
- The `user` field in the request JSON is set to `session_id` for Dify's user-scoped logging.
