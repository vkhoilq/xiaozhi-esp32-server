# Intent LLM Provider

## Responsibility

Performs LLM-powered intent classification on user utterances to determine whether the user wants to continue chatting, request context information (time/date), execute a function call (music playback, device control), or exit the conversation.

## Design

- **`class IntentProvider(IntentProviderBase)`**: Full LLM-based intent detection
- **Dynamic System Prompt Generation**: `get_intent_system_prompt(functions_list)` builds a comprehensive prompt from:
  - Registered function definitions (name, description, parameters)
  - MCP (Model Context Protocol) tool definitions (if available)
  - Rules for special cases: time/date/location queries → `result_for_context`, exit intent → `handle_exit_intent`, etc.
  - Strict JSON-only output requirement
- **Result Caching**: Uses `cache_manager` with `CacheType.INTENT` and MD5 hash of `(device_id + text)` as key
- **Context Augmentation**: Before calling LLM, appends:
  - Music file names from `initialize_music_handler()` for music playback intent
  - Home Assistant device list (if configured) for smart home control
- **Dialogue Window**: Uses last 4 dialogue turns (configurable via `self.history_count`) for context
- **Functions Call Handling**: Supports both single `function_call` and array `function_calls` formats
- **Response Parsing**: Regex-extracts JSON from LLM response, handles malformed output gracefully
- **Dialogue Cleanup**: On `continue_chat`, removes tool/function messages from dialogue history to prevent context pollution

## Flow

1. `detect_intent(conn, dialogue_history, text)` called
2. Returns `continue_chat` early if no `func_handler`
3. Checks intent cache → return cached if hit
4. Generates system prompt (lazy-loaded, cached in `self.promot`)
5. Appends music names and Home Assistant device context to prompt
6. Builds user prompt with recent dialogue history
7. Calls `self.llm.response_no_stream(system_prompt, user_prompt)`
8. Extracts JSON from response → parses intent
9. Handles special intents (`result_for_context`, `continue_chat`, function calls)
10. Caches result, returns intent JSON string

## Integration

- **Dependency**: `llm` — any provider implementing `response_no_stream()` (OpenAI-compatible)
- **Dependency**: `conn.func_handler.get_functions()` — returns list of available function definitions
- **Dependency**: `conn.mcp_client.get_available_tools()` — optional MCP tools
- **Dependency**: `core.utils.cache.manager` — intent caching
- **Dependency**: `plugins_func.functions.play_music.initialize_music_handler()` — provides music file names
- **Consumer**: Intent JSON drives function dispatch in the main chat pipeline
