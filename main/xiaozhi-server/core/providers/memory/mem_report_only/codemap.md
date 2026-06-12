# mem_report_only Memory Provider (`core/providers/memory/mem_report_only/`)

## Responsibility
A **no-op / passthrough memory provider** that explicitly performs no memory summarization or storage. Chat history is still sent to the Java management API for reporting/analytics purposes, but no local or cloud memory is persisted. The class name and comments indicate this is "仅上报聊天记录，不进行记忆总结" (report only, no memory summarization).

## Design
- **Null Object Pattern**: All methods are implemented as no-ops that log debug messages and return empty values. This satisfies the `MemoryProviderBase` ABC contract without side effects.
- **Lightweight**: Minimal state — constructor takes config but stores nothing.

## Flow
1. **`__init__(config, summary_memory=None)`**: Calls super constructor, does nothing else.
2. **`save_memory(msgs, session_id)`**: Logs debug message, returns `None`.
3. **`query_memory(query)`**: Logs debug message, returns `""`.

## Integration
- **Dependencies**: `config.logger`, `..base.MemoryProviderBase`
- **Configuration** (in `config.yaml`):
  ```yaml
  selected_module:
    Memory: "mem_report_only"
  Memory:
    mem_report_only: {}
  ```
- **Consumers**: Used when the system should function completely without memory retrieval, while the Java backend still receives raw chat history for reporting.
