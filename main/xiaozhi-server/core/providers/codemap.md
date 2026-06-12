# core/providers/

## Responsibility

Pluggable provider system for all AI and voice services. Each subdirectory implements a specific capability (ASR, TTS, LLM, VAD, Memory, Intent, VLLM, Tools) using a **Strategy pattern** with a common base class and multiple swappable adapters. The `selected_module` config key determines which concrete implementation is instantiated at runtime via factory functions in `core/utils/`.

## Design

- **Strategy + Factory Method pattern**: Each provider type has an abstract base class (`ASRProvider`, `TTSProvider`, `LLMProvider`, etc.) defining the interface, and multiple concrete implementations selected by config name.
- **Dynamic import**: Factory functions in `core/utils/` (e.g., `asr.create_instance()`, `tts.create_instance()`) dynamically import `core/providers/{type}/{config_name}.py` and instantiate the expected class.
- **Shared instances**: Some providers (VAD, local ASR) are shared across all connections; others (LLM, TTS, Memory) are per-connection.
- **Streaming-first**: Most providers support streaming (async generators) for real-time audio/text delivery.

## Flow

1. `config.yaml` defines `selected_module: { ASR: FunASR, LLM: ChatGLMLLM, TTS: EdgeTTS, ... }`.
2. `initialize_modules()` in `core/utils/modules_initialize.py` reads config → calls each factory.
3. Factory: `asr.create_instance("fun_local", config["ASR"]["FunASR"])` → imports `core.providers.asr.fun_local` → returns `ASRProvider(config)`.
4. Per-connection: `ConnectionHandler` receives shared + per-connection provider instances.
5. During chat: audio → VAD → ASR → Intent → LLM → Tools → TTS → Opus → device.

## Integration

- **Consumed by**: `core/connection.py` (per-connection lifecycle), `core/utils/modules_initialize.py` (initialization), `core/utils/` factory functions (instantiation).
- **Subdirectories**: `asr/`, `tts/`, `llm/`, `vad/`, `memory/`, `intent/`, `vllm/`, `tools/` — each with its own base class and adapters.
- **Config-driven**: All provider selection is driven by `config.yaml` → `selected_module` keys.