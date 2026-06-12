# Intent NoIntent Provider

## Responsibility

A pass-through intent detection implementation that always returns "continue chatting," effectively disabling intent-based routing. All user utterances flow directly to the main chat LLM without any pre-classification.

## Design

- **Minimal Implementation**: `IntentProvider` extends `IntentProviderBase` and overrides `detect_intent()` to unconditionally return:

  ```json
  {"function_call": {"name": "continue_chat"}}
  ```

- No LLM query, no function analysis, no caching.
- Used when intent detection is explicitly disabled in configuration.

## Flow

1. `detect_intent(conn, dialogue_history, text)` called
2. Logs debug message "Using NoIntentProvider, always returning continue chat"
3. Returns fixed JSON indicating `continue_chat`

## Integration

- **Config key**: Set `intent` in config to `nointent` to use this provider
- **Consumer**: Downstream pipeline always receives `continue_chat` intent, proceeds with normal LLM conversation without any function-call routing
