# Intent Provider

## Responsibility

Analyzes user utterances (text from ASR) to determine the user's intent before dispatching to the main LLM chat pipeline. Intent detection enables function calling (e.g., music playback, weather queries, device control) and determines whether the system should continue chatting, exit, or execute a specific action.

## Design

- **Abstract Base Class**: `IntentProviderBase` defines:
  - `__init__(config)` — stores config reference
  - `set_llm(llm)` — binds an LLM provider for intent-aware implementations
  - `detect_intent(conn, dialogue_history, text) -> str` — abstract method returning a JSON string with intent

- **Three Implementations**:

  1. **NoIntent** (`nointent/nointent.py`):
     - Always returns `{"function_call": {"name": "continue_chat"}}`
     - Used when intent detection is disabled; all utterances pass through to normal chat

  2. **FunctionCall** (`function_call/function_call.py`):
     - Always returns `{"function_call": {"name": "continue_chat"}}`
     - Identical behavior to NoIntent; preserved for compatibility with function_call-based routing

  3. **IntentLLM** (`intent_llm/intent_llm.py`):
     - Uses an LLM (via `self.llm.response_no_stream()`) to classify intent
     - Generates a dynamic system prompt based on available functions (from `func_handler` and MCP tools)
     - Includes music file names and Home Assistant device list in context
     - Caches intent results by `(device_id, text)` MD5 hash to reduce LLM calls
     - Supports result_for_context (time/date queries), continue_chat, handle_exit_intent, and arbitrary function calls
     - Cleans LLM response by extracting JSON with regex, handles parse errors gracefully

## Flow (IntentLLM)

1. `detect_intent(conn, dialogue_history, text)` called with latest user text
2. Returns early `continue_chat` if no `func_handler` is set
3. Checks intent cache; returns cached result if found
4. Builds system prompt from registered functions + music names + Home Assistant devices
5. Constructs user prompt with recent dialogue history (last 4 turns by default)
6. Calls `llm.response_no_stream(system_prompt, user_prompt)` with the built prompts
7. Parses LLM output: regex-extracts JSON, handles function_call/function_calls formats
8. For `result_for_context`: directly returns (context-based answer)
9. For `continue_chat`: cleans dialogue history of tool/function messages
10. For other function names: returns JSON for function dispatch
11. Caches intent result and returns

## Integration

- **Consumer**: Core chat pipeline calls `intent_provider.detect_intent()` before invoking the main LLM
- **Consumer**: Intent result drives function dispatch (music, weather, Home Assistant, etc.)
- **Dependency**: `llm` provider (set via `set_llm()`) — any OpenAI-compatible LLM
- **Dependency**: `conn.func_handler` — provides available function definitions
- **Dependency**: `conn.mcp_client` — optional MCP tool integration
- **Dependency**: `plugins_func.functions.play_music.initialize_music_handler()` — for music context
- **Dependency**: `core.utils.cache.manager` — intent result caching
