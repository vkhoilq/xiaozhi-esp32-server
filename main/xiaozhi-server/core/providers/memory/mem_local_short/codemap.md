# mem_local_short Memory Provider (`core/providers/memory/mem_local_short/`)

## Responsibility
Provides **local, LLM-driven conversational memory summarization**. Instead of storing raw messages, it uses a prompt-engineered LLM call to distill conversation history into a structured JSON memory object ("时空记忆编织者"). Persists the summary to a local YAML file (`data/.memory.yaml`), keyed by `role_id`. Designed for scenarios where privacy or offline operation is preferred over cloud APIs.

## Design
- **LLM-as-a-Service Architecture**: The provider uses the injected LLM (set via `set_llm`) to generate a structured memory summary. The `short_term_memory_prompt` is a detailed system prompt defining a "时空记忆编织者" (Space-Time Memory Weaver) with explicit JSON schema, evaluation dimensions (timeliness 40%, emotional intensity 35%, association density 25%), and space-optimization rules (淘汰预警 at 900 characters).
- **File-Based Persistence**: Uses `yaml` library to read/write `data/.memory.yaml`, with `role_id` as the key. Supports `save_to_file=False` mode for API-based summarization (Java backend).
- **Two-Phase Initialization**: `__init__` calls `load_memory(summary_memory)`, which either accepts pre-loaded summary from API or reads from local YAML. `init_memory` is overridden to accept `summary_memory` and `save_to_file` parameters.
- **JSON Extraction Utility**: `extract_json_data()` handles the common pattern of LLMs returning JSON inside markdown code fences (```json ... ```).

## Flow
1. **`save_memory(msgs, session_id)`**:
   - Builds a text prompt from user/assistant messages + existing `short_memory` + current timestamp
   - Calls `self.llm.response_no_stream(prompt, temperature=0.2, max_tokens=2000)` via the injected LLM
   - Extracts JSON from the LLM response, validates with `json.loads`
   - Saves to `data/.memory.yaml` via `save_memory_to_file()`
   - If `save_to_file=False`, calls `generate_and_save_chat_summary()` on the Java management API
2. **`query_memory(query)`**: Returns the current `self.short_memory` string directly (no semantic search — purely sequential summarization).
3. **`load_memory(summary_memory)`**: Either accepts an externally provided summary or reads from `data/.memory.yaml` by `role_id`.
4. **`save_memory_to_file()`**: Reads existing YAML, updates the entry for current `role_id`, writes back.

## Integration
- **Dependencies**: `yaml`, `json`, `time`, `os`, `config.config_loader`, `config.manage_api_client` (for Java API summarization), `core.utils.util.check_model_key`
- **Configuration** (in `config.yaml`):
  ```yaml
  selected_module:
    Memory: "mem_local_short"
  Memory:
    mem_local_short:
      # No special config required
  ```
- **File Output**: `data/.memory.yaml` — a flat mapping of `role_id → JSON memory string`
- **Consumers**: Dialogue processing layer calls `save_memory` after each turn; LLM response generation calls `query_memory` before constructing context.
- **Important**: Requires an LLM instance with `response_no_stream` method. The LLM API key is validated separately via `check_model_key`.
