# Intent FunctionCall Provider

## Responsibility

A no-op intent detection implementation that always signals "continue chatting." Used as a fallback or when intent-based dispatching is delegated entirely to the main LLM pipeline.

## Design

- **Minimal Implementation**: `IntentProvider` extends `IntentProviderBase` and overrides `detect_intent()` to unconditionally return:

  ```json
  {"function_call": {"name": "continue_chat"}}
  ```

- No LLM query, no analysis, no function call matching.
- Preserved in the codebase to support systems that expect a function_call-style response format even when disabling intent detection.

## Flow

1. `detect_intent(conn, dialogue_history, text)` called
2. Logs debug message "Using functionCallProvider, always returning continue chat"
3. Returns fixed JSON indicating `continue_chat`

## Integration

- **Config key**: Set `intent` in config to use this provider
- **Consumer**: Downstream pipeline receives `continue_chat` intent, proceeds with normal LLM conversation
