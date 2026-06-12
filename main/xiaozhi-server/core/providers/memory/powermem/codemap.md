# PowerMem Memory Provider (`core/providers/memory/powermem/`)

## Responsibility
Integrates the **open-source PowerMem agent memory component** (from OceanBase, https://github.com/oceanbase/powermem) as a memory backend. Supports multiple storage backends (sqlite, oceanbase, postgres), LLM providers (qwen, openai), and embedding providers. Provides two operational modes: standard `AsyncMemory` (general conversation memory) and `UserMemory` (user profiling mode, requires OceanBase).

## Design
- **SDK Wrapper Pattern**: Wraps the `powermem` Python SDK (either `AsyncMemory` or `UserMemory`), delegating all storage and retrieval to the PowerMem component.
- **Dual-Mode Architecture**:
  - **AsyncMemory mode** (`enable_user_profile=False`): Standard semantic memory — messages are stored and queried via vector similarity search.
  - **UserMemory mode** (`enable_user_profile=True`): User profiling — builds and maintains a user profile in addition to general memories. Profile is cached in `last_profile_content` for fast access (cache-first strategy).
- **Graceful Degradation**: If `powermem` package is not installed, config is invalid, or initialization fails, sets `self.use_powermem = False` and no-ops all operations.
- **Flexible Configuration**: Supports both PowerMem-native config keys (`database`, `llm`, `embedder`) and Mem0-compatible config keys (`vector_store`, `llm`, `embedder`) for interoperability.
- **JSON Content Extraction**: Same pattern as mem0ai — extracts `"content"` from JSON-wrapped ASR payloads.
- **Sync/Async Bridge**: `save_memory` handles both sync and async return values from PowerMem SDK methods via `asyncio.iscoroutine` check.

## Flow
1. **Initialization**:
   - Reads config for `enable_user_profile`, database/LLM/embedding providers
   - Builds a `powermem_config` dictionary, resolving provider-specific base URLs (qwen→dashscope_base_url, openai→openai_base_url)
   - Imports and instantiates `UserMemory` or `AsyncMemory` based on `enable_user_profile`
   - Logs initialization result with mode and provider info
2. **`save_memory(msgs, session_id)`**:
   - Filters system messages, extracts JSON content
   - Calls `self.memory_client.add(messages=messages, user_id=self.role_id)`
   - In UserMemory mode, caches any extracted `profile_content` from the result
3. **`query_memory(query)`**:
   - Extracts search query from JSON payload if needed
   - In UserMemory mode, fetches user profile via `get_user_profile()` (cache-first)
   - Calls `self.memory_client.search(query=search_query, user_id=self.role_id, limit=30)`
     - UserMemory uses sync search via `asyncio.to_thread`
     - AsyncMemory uses native async search
   - Formats results with timestamp prefixes, sorted newest-first
   - Combines profile section + relevant memories section
4. **`get_user_profile()`**:
   - Cache-first: returns `last_profile_content` if non-empty
   - Cache miss: calls `self.memory_client.profile(role_id)` via `asyncio.to_thread`
   - Extracts `profile_content` first, falls back to `topics` JSON serialization
   - Caches and returns the result

## Integration
- **Dependencies**: `powermem` (PyPI package, optional), `asyncio`, `json`, `traceback`
- **Configuration** (in `config.yaml`):
  ```yaml
  selected_module:
    Memory: "powermem"
  Memory:
    powermem:
      enable_user_profile: false                # Enable UserMemory (requires OceanBase)
      database_provider: sqlite                  # sqlite | oceanbase | postgres
      llm_provider: qwen                         # qwen | openai
      embedding_provider: qwen                   # qwen | openai
      vector_store:                              # Optional: override database config
        provider: sqlite
        config: {}
      llm:                                       # Optional: override LLM config
        provider: qwen
        config:
          api_key: "..."
          model: "qwen-plus"
      embedder:                                  # Optional: override embedding config
        provider: qwen
        config:
          api_key: "..."
          model: "text-embedding-v3"
  ```
- **Consumers**: Dialogue/connection handling layer calls `save_memory` and `query_memory` during conversation processing.
