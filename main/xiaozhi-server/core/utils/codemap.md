# core/utils/

## Responsibility

Shared utility modules that support the core runtime. This is the largest utility directory, providing factory functions for all pluggable providers, caching infrastructure, audio processing, dialogue management, prompt rendering, and text/emoji utilities. Contains:

### Provider Factories (Factory Method Pattern)
- **`tts.py`**: `create_instance(class_name, ...)` — dynamically imports and instantiates TTS providers from `core/providers/tts/{name}.py`. Also provides `MarkdownCleaner` (regex-based markdown → plain text for TTS) and `convert_percentage_to_range()`.
- **`asr.py`**: `create_instance(class_name, ...)` — dynamically imports ASR providers from `core/providers/asr/{name}.py`.
- **`llm.py`**: `create_instance(class_name, ...)` — dynamically imports LLM providers from `core/providers/llm/{name}/{name}.py`.
- **`vad.py`**: `create_instance(class_name, ...)` — dynamically imports VAD providers from `core/providers/vad/{name}.py`.
- **`vllm.py`**: `create_instance(class_name, ...)` — dynamically imports VLLM (vision) providers from `core/providers/vllm/{name}.py`.
- **`intent.py`**: `create_instance(class_name, ...)` — dynamically imports Intent providers from `core/providers/intent/{name}/{name}.py`.
- **`memory.py`**: `create_instance(class_name, ...)` — dynamically imports Memory providers from `core/providers/memory/{name}/{name}.py`.

### Orchestration
- **`modules_initialize.py`**: `initialize_modules()` — centralized initialization of all module types (VAD, ASR, LLM, TTS, Memory, Intent) based on config's `selected_module`. Also provides `initialize_tts()`, `initialize_asr()`, `initialize_voiceprint()`.

### Audio Processing
- **`util.py`**: Large general utility file — `audio_to_data()` (file → Opus frames with caching), `opus_datas_to_wav_bytes()`, `pcm_to_data_stream()`, `audio_bytes_to_data_stream()`, `audio_to_data_stream()`, `get_local_ip()`, `get_ip_info()`, `is_private_ip()`, `check_ffmpeg_installed()`, `extract_json_from_string()`, `remove_punctuation_and_length()`, `filter_sensitive_info()`, `get_vision_url()`, `is_valid_image_file()`, `validate_mcp_endpoint()`, `get_system_error_response()`, `check_vad_update()`, `check_asr_update()`.
- **`audioRateController.py`** (`AudioRateController`): Frame-accurate (60ms) audio packet scheduler with pre-buffer, dynamic timestamp reset on pause/resume, and async event-driven queue draining.
- **`opus_encoder_utils.py`** (`OpusEncoderUtils`): PCM → Opus streaming encoder using `opuslib_next` with frame buffering and validation.
- **`p3.py`**: `.p3` file format decoder — reads 4-byte header + Opus payload chunks from files or bytes.

### Dialogue & Prompt
- **`dialogue.py`** (`Message`, `Dialogue`): In-memory conversation history manager. Messages have role, content, tool_calls, tool_call_id, and is_temporary flag. `get_llm_dialogue_with_memory()` builds the LLM request array with four segments: (1) static system prompt, (2) few-shot examples, (3) dynamic context (time, memory, speakers), (4) actual conversation. Handles dangling tool_calls repair.
- **`prompt_manager.py`** (`PromptManager`): Jinja2-based system prompt rendering. Loads `agent-base-prompt.txt`, injects context variables (time, date, lunar date, location, weather, speaker info, dynamic context from external APIs), caches device-specific prompts, and supports emoji toggling.
- **`textUtils.py`**: Emoji detection/mapping (`EMOJI_MAP`, `is_emoji()`), punctuation stripping, emotion extraction from text (sends `{"type": "llm", "text": emoji, "emotion": ...}` to client).

### Caching (see `cache/` subdirectory)
- **`cache/`**: `GlobalCacheManager` with TTL/LRU/FIXED_SIZE strategies for config, location, weather, IP info, audio data, voiceprint health, device prompts.

### Other Utilities
- **`current_time.py`**: Time utilities — `get_current_time()`, `get_current_date()`, `get_current_weekday()` (Chinese), `get_current_lunar_date()` (via `cnlunar`).
- **`output_counter.py`**: Per-device daily output character counter with auto-reset at midnight.
- **`gc_manager.py`** (`GlobalGCManager`): Singleton that runs `gc.collect()` every 300 seconds in a thread pool to prevent GIL contention.
- **`wakeup_word.py`** (`WakeupWordsConfig`): YAML-backed persistent store for wake word TTS responses, with file-lock protection and MD5-based voice hashing.
- **`voiceprint_provider.py`** (`VoiceprintProvider`): Speaker identification client — connects to external voiceprint API, manages speaker config, health checks with caching, and async audio-based speaker identification.
- **`context_provider.py`** (`ContextDataProvider`): Fetches dynamic context data (health, stocks, etc.) from configurable HTTP APIs for injection into the LLM prompt.
- **`auth.py`** (`AuthToken`): JWT + AES-GCM token generation/verification for the Vision API (distinct from the WebSocket HMAC auth in `core/auth.py`).

## Design

- **Factory Method pattern**: All provider factories follow the same pattern — check file existence in `core/providers/{type}/`, dynamic import, and instantiate the expected class name (`ASRProvider`, `TTSProvider`, `LLMProvider`, etc.).
- **Two auth systems**: `AuthToken` (JWT + AES-GCM) for HTTP Vision API; `AuthManager` (HMAC-SHA256) for WebSocket connections. Different security profiles for different endpoints.
- **Four-segment dialogue construction**: `get_llm_dialogue_with_memory()` separates static system prompt → few-shot examples → dynamic context → actual conversation to maximize prefix caching in LLM APIs.
- **Cache-first data access**: Location, weather, IP info, audio data, device prompts, and voiceprint health all flow through `GlobalCacheManager` with type-specific TTLs.
- **Thread pool delegation**: Heavy synchronous operations (audio encoding, GC, LLM calls) are offloaded to `ThreadPoolExecutor` or `run_in_executor()` to avoid blocking the async event loop.

## Flow

### Provider Instantiation
```
config["selected_module"]["LLM"] = "ChatGLMLLM"
  → modules_initialize.initialize_modules()
    → llm.create_instance("openai", config["LLM"]["ChatGLMLLM"])
      → import core.providers.llm.openai.openai
        → return LLMProvider(config)  # the OpenAI-compatible wrapper
```

### Audio Encoding Flow
```
audio file → util.audio_to_data(path, is_opus=True)
  → check cache → if miss: AudioSegment.from_file() → PCM → opuslib_next.Encoder → frames
  → store in cache → return list[bytes]
```

### Dialogue Construction Flow
```
conn.dialogue.get_llm_dialogue_with_memory(memory_str, voiceprint_config)
  → segment 1: static system prompt (prefix cacheable)
  → segment 2: few-shot examples (tool call demonstrations)
  → segment 3: dynamic system (time, memory, speakers)
  → segment 4: actual conversation history
  → repair dangling tool_calls → return list[dict]
```

## Integration

- **Consumed by**: All `core/` modules (`connection.py`, `websocket_server.py`, `handle/`), all `core/providers/` (indirectly through factory functions).
- **Depends on**: `config/` (logger, cache manager), external APIs (IP geolocation at `whois.pconline.com.cn`, voiceprint server, context data providers), `cnlunar` (lunar calendar), `opuslib_next` (Opus codec), `pydub` (audio format conversion), `pyjwt` + `cryptography` (token auth), `jinja2` (template rendering), `httpx` / `aiohttp` (HTTP requests).
