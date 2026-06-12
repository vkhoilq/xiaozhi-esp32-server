# nomem Memory Provider (`core/providers/memory/nomem/`)

## Responsibility
A **complete no-op memory provider** that disables all memory functionality. Unlike `mem_report_only`, this provider does not even imply reporting — it is the explicit choice to run without any memory subsystem at all. Comments in the source state: "不使用记忆，可以选择此模块" (no memory, you can choose this module).

## Design
- **Null Object Pattern**: All methods are no-ops. Constructor takes config but ignores it.
- **Minimal Overhead**: The lightest possible implementation of `MemoryProviderBase`.

## Flow
1. **`__init__(config, summary_memory=None)`**: Calls super constructor.
2. **`save_memory(msgs, session_id)`**: Logs "nomem mode" debug message, returns `None`.
3. **`query_memory(query)`**: Logs "nomem mode" debug message, returns `""`.

## Integration
- **Dependencies**: `config.logger`, `..base.MemoryProviderBase`
- **Configuration** (in `config.yaml`):
  ```yaml
  selected_module:
    Memory: "nomem"
  Memory:
    nomem: {}
  ```
- **Consumers**: Useful for testing, minimal deployments, or privacy-sensitive scenarios where no conversation data should be stored or summarized anywhere.
