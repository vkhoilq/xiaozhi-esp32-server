# core/handle/

## Responsibility

Message handling and routing layer for inbound WebSocket text messages and audio processing. This directory translates raw protocol messages from ESP32 devices into domain-specific actions. It contains:

- **`textHandle.py`**: Entry point — `handleTextMessage(conn, message)` delegates to the `TextMessageProcessor`.
- **`textMessageProcessor.py`** (`TextMessageProcessor`): Parses JSON text messages, extracts `type` field, dispatches to registered handler via `TextMessageHandlerRegistry`.
- **`textMessageHandlerRegistry.py`** (`TextMessageHandlerRegistry`): Registry of `TextMessageHandler` implementations keyed by `TextMessageType` enum value. Auto-registers all default handlers on construction.
- **`textMessageHandler.py`** (`TextMessageHandler`): Abstract base class with `handle(conn, msg_json)` and `message_type` property.
- **`textMessageType.py`** (`TextMessageType`): Enum of supported message types: `HELLO`, `ABORT`, `LISTEN`, `IOT`, `MCP`, `SERVER`, `PING`.
- **`helloHandle.py`**: Handles the initial `hello` handshake — configures audio params, detects client features (MCP support, emoji), sends welcome message. Also implements wake-word detection and response caching (`WakeupWordsConfig`, `checkWakeupWords`, `wakeupWordsResponse`).
- **`abortHandle.py`**: Handles `abort` — interrupts LLM/TTS output, clears audio queues, sends stop signal to client.
- **`listenHandle.py`** (via `listenMessageHandler.py`): Handles `listen` state changes — manages VAD detection mode, triggers ASR on voice stop, processes `detect` states with wake-word checking and [device_call] routing.
- **`intentHandler.py`**: Intent analysis pipeline — checks exit commands → wake words → LLM-based intent analysis (for `intent_llm` mode) or function call routing (for `function_call` mode). Processes `ActionResponse` results (RESPONSE, REQLLM, NOTFOUND, ERROR).
- **`receiveAudioHandle.py`**: Audio processing entry point — VAD voice activity detection, idle timeout monitoring (`no_voice_close_connect`), device binding prompts, daily output limit enforcement, and triggers `startToChat()`.
- **`sendAudioHandle.py`**: TTS audio output — `AudioRateController`-based timed sending of Opus frames, pre-buffer logic, MQTT gateway packet formatting, state message delivery (sentence_start/stop, STT text, display messages).
- **`reportHandle.py`**: Chat history reporting — queues ASR/TTS data and tool calls for async upload to the manager API. Converts Opus to WAV for audio attachment.

## Design

- **Strategy + Registry pattern**: `TextMessageHandler` defines the interface; `TextMessageHandlerRegistry` maps `TextMessageType` → handler. New message types are added by implementing `TextMessageHandler` and registering in the registry.
- **Two parallel processing paths**: Audio bytes go through `receiveAudioHandle` → VAD → ASR → `startToChat()`. Text strings go through `textHandle` → JSON parse → type-based handler dispatch.
- **Intent routing**: Three intent modes supported — `nointent` (pass-through to LLM), `intent_llm` (separate LLM call for intent classification), `function_call` (LLM-native tool calling with `direct_answer` virtual tool injected as a routing mechanism).
- **Wake word caching**: `WakeupWordsConfig` persists generated TTS responses for wake words to `data/.wakeup_words.yaml` with file-lock protection, caching per-voice audio files to avoid regenerating on every wake.
- **Audio rate control**: `AudioRateController` in `sendAudioHandle` provides precise 60ms-frame pacing with pre-buffer (5 packets), dynamic timing, and pause-resume timestamp reset.
- **Queued reporting**: `reportHandle` implements a producer-consumer pattern with per-connection `queue.Queue` and a daemon thread that drains the queue to the manager API.

## Flow

### Text Message Flow
```
websocket message (str)
  → textHandle.handleTextMessage(conn, message)
    → TextMessageProcessor.process_message(conn, message)
      → json.loads(message) → extract "type"
        → registry.get_handler(type) → handler.handle(conn, msg_json)
```

### Audio Message Flow
```
websocket message (bytes)
  → ConnectionHandler._route_message()
    → asr_audio_queue.put(message)
      → ASR provider processes
        → voice_stop detected
          → startToChat(conn, text)
            → handle_user_intent() → check exit → wake words → intent analysis
              → if intent handled: done
              → else: conn.chat(text) → LLM → TTS → sendAudioHandle
```

### Reporting Flow
```python
enqueue_asr_report(conn, text, opus_data)  # or enqueue_tts_report / enqueue_tool_report
  → conn.report_queue.put((type, text, audio, timestamp))
    → _report_worker thread
      → report(conn, type, text, opus_data, timestamp)
        → manage_api_client.report()
```

## Integration

- **Consumed by**: `core/connection.py` — imports and calls `handleTextMessage`, `handleAudioMessage`, `startToChat`, reporting functions.
- **Depends on**: `core/providers/` (VAD, ASR, TTS providers), `core/utils/` (dialogue, textUtils, audioRateController, output_counter, wakeup_word, util), `core/handle/textHandler/` (per-type message handler implementations), `plugins_func/` (function plugins, Action/ActionResponse).
- **External**: Reporting goes to the Java manager API via `config/manage_api_client.py`. Wake word responses can be cached locally. Device location is looked up via `whois.pconline.com.cn`.
