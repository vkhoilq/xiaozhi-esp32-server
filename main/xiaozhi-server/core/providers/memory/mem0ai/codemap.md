# mem0ai Memory Provider (`core/providers/memory/mem0ai/`)

## Responsibility
Implements memory persistence via the **Mem0 cloud API** (`mem0` Python SDK). Stores conversation messages as structured user memories on Mem0's hosted service, and performs semantic similarity searches to retrieve relevant past memories. Designed as an external-cloud backed memory strategy.

## Design
- **Wrapper Pattern**: The `MemoryProvider` class wraps the `MemoryClient` SDK (`from mem0 import MemoryClient`), delegating all storage and retrieval to the external service.
- **Graceful Degradation**: If the API key is missing/invalid or connection fails, sets `self.use_mem0 = False` and silently no-ops all operations — the system continues without memory.
- **JSON Content Extraction**: Both `save_memory` and `query_memory` check if message content is JSON (`{...}`) and extract the `"content"` field, supporting ASR annotations (emotion/language tags).
- **Timestamp Sorting**: Query results are sorted descending by `updated_at` so the most recent memories are presented first.

## Flow
1. **Initialization**: Config is read for `api_key` and `api_version` → validates key via `check_model_key` → creates `MemoryClient` instance → sets `use_mem0` flag.
2. **`save_memory(msgs, session_id)`**:
   - Filters out `system` and `tool` role messages
   - Extracts plain content from JSON-wrapped ASR payloads
   - Calls `self.client.add(messages, user_id=self.role_id)` on the Mem0 API
3. **`query_memory(query)`**:
   - Extracts search query from JSON payload if needed
   - Calls `self.client.search(search_query, filters={"user_id": self.role_id})`
   - Formats results as `[YYYY-MM-DD HH:MM:SS] memory_text` lines sorted newest-first
   - Returns empty string on failure or no results

## Integration
- **Dependencies**: `mem0` (PyPI package), `core.utils.util.check_model_key`, `config.logger`
- **Configuration** (in `config.yaml`):
  ```yaml
  selected_module:
    Memory: "mem0ai"
  Memory:
    mem0ai:
      api_key: "your_mem0_api_key"
      api_version: "v1.1"
  ```
- **Consumers**: Called by `core.handle` dialogue processing after conversation turns and before LLM response generation.
