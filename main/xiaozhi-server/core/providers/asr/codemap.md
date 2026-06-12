# ASR (Automatic Speech Recognition) Provider

## Responsibility

Converts incoming audio streams (Opus/PCM) from ESP32 clients into text transcripts using a pluggable provider architecture. Supports both streaming (real-time, per-frame) and non-streaming (batch/end-of-utterance) recognition modes, as well as local on-device inference. Each utterance is submitted to downstream intent processing via `startToChat()`.

## Design

- **Abstract Base Class**: `ASRProviderBase` (in `base.py`) defines the contract:
  - `speech_to_text(opus_data, session_id, audio_format, artifacts)` — core abstract method
  - `receive_audio(conn, audio, audio_have_voice)` — receives per-frame audio, buffers it, and triggers recognition based on VAD state or manual mode
  - `handle_voice_stop(conn, asr_audio_task)` — called when VAD detects speech end; decodes Opus→PCM, optionally runs voiceprint identification in parallel, then calls `speech_to_text_wrapper()`
  - `speech_to_text_wrapper()` — handles Opus→PCM conversion, temp file creation, disk-space checks, and cleanup
  - `decode_opus(opus_data)` — static method using `opuslib_next.Decoder(16000, 1)` with 960-sample frames
  - `AudioArtifacts` NamedTuple — bundles PCM frames, merged PCM bytes, WAV file path, temp path
  - `requires_file()` / `prefers_temp_file()` / `build_temp_file()` — file-based ASR support
- **InterfaceType Enum** (`dto/dto.py`): `STREAM` (WebSocket streaming), `NON_STREAM` (REST batch), `LOCAL` (on-device)
- **Provider Discovery**: Each adapter module exports `class ASRProvider(ASRProviderBase)`; the server instantiates the correct one via config key
- **Threading Model**: A dedicated daemon thread (`asr_priority_thread`) drains `conn.asr_audio_queue` and calls `handleAudioMessage()` on the connection's event loop via `asyncio.run_coroutine_threadsafe()`
- **Audio Pipeline**: Opus packets arrive → decoded to PCM → buffered in `conn.asr_audio` → when VAD signals stop → sent to provider's `speech_to_text()`

## Flow

1. `receive_audio(conn, audio, audio_have_voice)` called per audio frame from WebSocket
2. Audio buffered in `conn.asr_audio` list
3. When `conn.client_voice_stop` is True (VAD detected end of speech) and `interface_type != STREAM`:
   - `handle_voice_stop()` is called with a copy of the buffer
   - Opus→PCM decoded → WAV prepared → voiceprint identification (optional) runs in parallel via `asyncio.gather()`
   - `speech_to_text_wrapper()` → `speech_to_text()` is called
   - Result checked for length → `enqueue_asr_report()` + `startToChat(conn, enhanced_text)` for downstream processing
4. For STREAM providers (aliyun_stream, aliyunbl_stream, doubao_stream, xunfei_stream):
   - `receive_audio()` overridden: on first voice, opens WebSocket to cloud ASR service, sends cached audio, then streams decoded PCM frames
   - A `_forward_results()` background task receives recognition results from the cloud
   - When sentence-end/final result arrives, calls `handle_voice_stop()` with accumulated audio
5. For file-based providers (openai, sherpa_onnx_local, qwen3_asr_flash): `requires_file()` returns True; the wrapper saves audio to a WAV file before calling `speech_to_text()`

## Integration

- **Consumer**: `core.handle.receiveAudioHandle.startToChat()` — receives the ASR text output and triggers the chat/LLM pipeline
- **Consumer**: `core.handle.reportHandle.enqueue_asr_report()` — logs/analytics reporting
- **Consumer**: `core.handle.receiveAudioHandle.handleAudioMessage()` — processes the audio queue entries
- **Dependency**: `core.providers.asr.dto.dto.InterfaceType` — used to distinguish STREAM vs NON_STREAM vs LOCAL behavior
- **Dependency**: Optional voiceprint provider runs in parallel for speaker identification
- **Configuration**: Provider-specific keys (api_key, appkey, model_dir, etc.) read from config dict; `output_dir` for saving audio files
- **Providers implemented**: Aliyun (REST), Aliyun Stream (WebSocket), Aliyun Bailian Stream (WebSocket), Baidu (REST), Doubao/Volcengine (REST), Doubao Stream (WebSocket), FunASR local, FunASR server (WebSocket), OpenAI Whisper (REST), Qwen3-ASR-Flash (DashScope), Sherpa-ONNX local, Tencent Cloud (REST), Vosk local, Xunfei Stream (WebSocket)
