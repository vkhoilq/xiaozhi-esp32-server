# core/providers/llm/coze/

## Responsibility

Provides an LLM provider implementation for **Coze** (扣子) — a conversational AI agent platform by ByteDance. This module bridges the voice assistant server with Coze's chat API via the official Python SDK (`cozepy`), enabling streaming dialogue completion with per-session conversation persistence.

## Design

### Class: `LLMProvider` (inherits `LLMProviderBase`)

**Initialization:**
- Reads `personal_access_token`, `bot_id`, and `user_id` from config.
- Validates the token via `check_model_key()`.
- Maintains `session_conversation_map` — a dict mapping the server's `session_id` → Coze `conversation_id`, enabling multi-turn continuity across user sessions.

**Key Design Decisions:**
- **Session-to-Conversation Mapping**: Each WebSocket session maps to a single Coze conversation. On first request for a session, a new conversation is created via `coze.conversations.create()`. Subsequent requests reuse the same conversation ID, preserving context.
- **User Message Extraction**: Uses `next(m for m in reversed(dialogue) if m["role"] == "user")` to extract only the last user utterance. Coze handles history internally via the conversation, so only the current user message is sent as `additional_messages`.
- **Streaming via SDK**: Uses `coze.chat.stream()` with the bot ID, user ID, conversation context, and the current user message. Iterates over `ChatEventType.CONVERSATION_MESSAGE_DELTA` events to yield incremental text.
- **Function Calling via Prompt Injection**: `response_with_functions()` serializes the available functions to JSON and prepends them using `get_system_prompt_for_function()` to the first user message when the dialogue has exactly 2 messages (system + user). Tool result messages (role="tool") are appended to the next user message.

## Flow

### Plain text flow (`response`)
1. Extract last user message from `dialogue`.
2. Look up or create a Coze conversation for `session_id`.
3. Call `coze.chat.stream()` with `bot_id`, `user_id`, `additional_messages=[user_msg]`, `conversation_id`.
4. Iterate over stream events; on `CONVERSATION_MESSAGE_DELTA`, yield the content delta.
5. Print each yielded chunk to stdout.

### Function-calling flow (`response_with_functions`)
1. If dialogue has exactly 2 messages and functions are provided: inject tool-use system prompt into the last user message.
2. If the last message has `role="tool"`: merge tool result into the preceding user message to simulate tool response.
3. Fall back to `response()`, yielding `(token, None)` (Coze does not support native function call streaming via this SDK path).

## Integration

### Dependencies
- `cozepy` (official Coze Python SDK)
- `core.providers.llm.base.LLMProviderBase` (parent class)
- `core.providers.llm.system_prompt.get_system_prompt_for_function()` (prompt injection utility)
- `config.logger` (structured logging)

### Consumed by
- The LLM provider factory/dispatch, invoked when config specifies `"type": "coze"` (or similar alias).
- Orchestration layer calls `response()` for dialogue completion or `response_with_functions()` for intent/tool resolution.

### Notable Details
- Uses `COZE_CN_BASE_URL` (China region base URL). For international deployments, the SDK can be configured with a different base URL.
- The `user_id` is a static config value, not the WebSocket session ID — Coze uses this for user-scoped bot configuration.
