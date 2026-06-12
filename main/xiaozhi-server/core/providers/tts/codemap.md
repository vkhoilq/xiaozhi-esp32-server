# TTS (Text-to-Speech) Provider

## Responsibility

Converts generated text responses from the LLM/chat pipeline into audio streams (Opus-encoded) sent to ESP32 clients. Supports streaming (per-sentence incremental audio), non-streaming (full audio file), and dual-stream (bidirectional WebSocket for concurrent text+audio) modes.

## Design

- **Abstract Base Class**: `TTSProviderBase` (in `base.py`) defines the contract:
  - `text_to_speak(text, output_file)` — core abstract method; returns audio bytes or saves to file
  - `to_tts_stream(text, opus_handler)` — high-level entry: cleans markdown, applies correct_words substitution, calls `text_to_speak()`, converts result to Opus data stream via `audio_bytes_to_data_stream()`, and pushes to `tts_audio_queue`
  - `to_tts(text)` — non-streaming variant, returns Opus data list or file path
  - `open_audio_channels(conn)` — creates Opus encoder, starts TTS processing thread and audio playout thread
  - `tts_text_priority_thread()` — drains `tts_text_queue`, segments text by punctuation, calls `to_tts_stream()` per segment, handles FILE content type
  - `_audio_play_priority_thread()` — drains `tts_audio_queue`, sends audio via `sendAudioMessage()`, handles TTS reporting
  - `correct_words` mechanism — configurable word substitution (e.g., abbreviations) with regex-based replacement and reverse mapping for display
  - `_get_segment_text()` — splits text by punctuation with sliding window for streaming TTS
  - `_match_stream_text()` — sliding-window replacement for cross-chunk word matching
  - `TTS_PARAM_CONFIG` — class-level list for percentage-based parameter scaling (volume, rate, pitch)
- **InterfaceType Enum** (`dto/dto.py`):
  - `NON_STREAM` — full audio generated at once (default)
  - `SINGLE_STREAM` — one streaming HTTP connection (index_stream, minimax_httpstream)
  - `DUAL_STREAM` — bidirectional WebSocket with separate text and audio channels (alibl_stream, aliyun_stream, huoshan_double_stream, xunfei_stream)
- **Threading Model**: Two daemon threads per connection:
  1. `tts_text_priority_thread` — consumes text from `tts_text_queue`
  2. `audio_play_priority_thread` — consumes audio from `tts_audio_queue`, sends to client via `sendAudioMessage()` on the event loop
- **Provider Discovery**: Each adapter exports `class TTSProvider(TTSProviderBase)`, selected by config
- **Audio Encoding**: Opus via `opus_encoder_utils.OpusEncoderUtils` with configurable sample rate (default 16000, index_stream uses 24000)

## Flow

1. `tts_text_priority_thread` receives `TTSMessageDTO` from `tts_text_queue`
2. Text message: segments by punctuation → per-segment calls `to_tts_stream()` → cleans markdown → applies correct_words → calls `text_to_speak()` → audio converted to Opus stream → put on `tts_audio_queue` with `SentenceType.FIRST/MIDDLE`
3. FILE message: audio file processed through `_process_audio_file_stream()` → Opus encoded → enqueued
4. LAST message: remaining text flushed → `SentenceType.LAST` marker enqueued
5. `_audio_play_priority_thread` reads from `tts_audio_queue`, calls `sendAudioMessage()` on the event loop, handles reporting via `enqueue_tts_report()`
6. For DUAL_STREAM providers: `start_session()` opens WebSocket, `_start_monitor_tts_response()` listens for audio frames, `finish_session()` sends termination

## Integration

- **Consumer**: `core.handle.sendAudioHandle.sendAudioMessage()` — sends audio frames to the WebSocket client
- **Consumer**: `core.handle.reportHandle.enqueue_tts_report()` — reports TTS metrics/analytics
- **Consumer**: `core.utils.output_counter.add_device_output()` — tracks output token usage per device
- **Dependency**: `core.utils.opus_encoder_utils` — Opus encoding
- **Dependency**: `core.utils.tts.MarkdownCleaner` — removes markdown formatting before TTS
- **Dependency**: `core.utils.tts.convert_percentage_to_range` — maps percentage config to ranges
- **Dependency**: `core.utils.textUtils` — punctuation/emoji filtering
- **Dependency**: `core.utils.p3` — Opus stream decode support
- **Providers implemented**: Aliyun (REST), Aliyun Stream (WebSocket), Aliyun Bailian Stream/CosyVoice (WebSocket), CozeCN, Custom (generic HTTP), Doubao/Volcengine, Edge TTS, FishSpeech, GPT-SoVITS v2, GPT-SoVITS v3, Huoshan Double Stream (Volcengine WebSocket), Index Stream, MiniMax HTTP Stream, OpenAI TTS, PaddleSpeech, SiliconFlow, Tencent Cloud, Xunfei Stream (WebSocket), Default (stub)

## Configuration

Each provider reads from a config dict with provider-specific keys. Common options:
- `voice` / `private_voice` — voice model/speaker
- `format` / `response_format` — audio encoding (wav, pcm, mp3)
- `output_dir` — where to save generated audio files
- `delete_audio_file` — whether to clean up after sending
- `correct_words` — list of "original|replacement" pairs
- `ttsVolume`, `ttsRate`, `ttsPitch` — percentage-based audio parameter tuning
