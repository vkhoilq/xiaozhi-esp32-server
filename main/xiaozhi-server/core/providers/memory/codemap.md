# Memory Providers Directory (`core/providers/memory/`)

## Responsibility
Defines the **abstract base contract** for all memory providers in the xiaozhi-esp32-server system. This directory establishes the `MemoryProviderBase` ABC that every concrete memory implementation must implement. It is the **pluggable persistence layer** for conversational memory — storing, summarizing, and querying historical dialogue context to enable long-term personalization across sessions.

## Design
- **Strategy Pattern**: The base class `MemoryProviderBase` (ABC) defines two abstract methods: `save_memory(msgs, session_id)` and `query_memory(query)`. Concrete providers (mem0ai, mem_local_short, powermem, nomem, mem_report_only) each implement their own strategy.
- **Template Method Pattern**: `init_memory(role_id, llm, **kwargs)` provides a default initialization that subclasses can override (e.g., mem_local_short extends it to load file-based memory).
- **Lazy LLM Injection**: `set_llm(llm)` allows late binding of an LLM instance, enabling providers to use LLM-based summarization (e.g., mem_local_short).
- **Provider Isolation**: Each provider is a self-contained subdirectory with a single `MemoryProvider` class, loaded dynamically via configuration.

## Flow
1. **Initialization**: `MemoryProviderBase.__init__(config)` stores config → `init_memory(role_id, llm)` stores identity → `set_llm(llm)` injects the LLM.
2. **Memory Storage**: `save_memory(msgs, session_id)` is called after each dialogue turn, receiving the full message list. The provider decides what to persist.
3. **Memory Retrieval**: `query_memory(query)` is called before LLM response generation to inject relevant context. Returns a formatted string.
4. **Consumers**: The dialogue/connection layer calls these two methods at specific points in the conversation lifecycle.

## Integration
- **Dependencies**: `config.logger` (logging), `abc` (abstract base class)
- **Consumers**: `core.handle` layers invoke memory operations during dialogue processing
- **Configuration**: Each provider is selected via `config["selected_module"]["Memory"]` with provider-specific config blocks
- **Subdirectory structure**: `mem0ai/`, `mem_local_short/`, `mem_report_only/`, `nomem/`, `powermem/` each contain a `MemoryProvider(MemoryProviderBase)` implementation
