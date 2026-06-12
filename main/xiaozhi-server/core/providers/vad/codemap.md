# VAD (Voice Activity Detection) Provider

## Responsibility

Detects whether incoming audio frames contain human speech, enabling the ASR pipeline to determine utterance boundaries (start and end of speech). Operates on per-frame Opus packets from the ESP32 client.

## Design

- **Abstract Base Class**: `VADProviderBase` defines single method:
  - `is_vad(conn, data) -> bool` — returns True if the audio frame contains speech

- **SileroVAD Implementation** (`silero.py`):
  - Uses `onnxruntime` with the Silero VAD model (`silero_vad.onnx`)
  - Employs **dual-threshold** logic: `vad_threshold` (default 0.5) and `vad_threshold_low` (default 0.2) for hysteresis
  - Audio is decoded from Opus to PCM (16kHz, mono) per connection using a per-connection decoder
  - Processes audio in 512-sample chunks with a 64-sample context window
  - Tracks a sliding window of `frame_window_threshold` (3) binary voice decisions
  - Measures silence duration in milliseconds (`min_silence_duration_ms`, default 1000) to detect utterance end
  - Per-connection state stored directly on the `conn` object (`_vad_state`, `_vad_context`, `_vad_opus_decoder`)

- **Manual Mode**: When `conn.client_listen_mode == "manual"`, `is_vad()` always returns True (ESP32 handles push-to-talk)

## Flow

1. `is_vad(conn, opus_packet)` called for each incoming audio frame
2. Opus packet decoded to PCM via per-connection decoder
3. PCM buffered in `conn.client_audio_buffer` (512-sample chunks)
4. Each chunk: int16→float32 normalization → concatenate with context → ONNX inference → speech probability output
5. Dual-threshold comparison: above `vad_threshold` → voice; below `vad_threshold_low` → silence; between → maintain previous state
6. Sliding window of 3 frames determines `client_have_voice`
7. If previously had voice and now silent → check `vad_last_voice_time` delta against `silence_threshold_ms` → set `conn.client_voice_stop = True`
8. Returns current voice activity boolean

## Integration

- **Consumer**: `ASRProviderBase.receive_audio()` uses VAD output (`audio_have_voice` and `conn.client_voice_stop`) to decide when to trigger ASR
- **Consumer**: Connection handler calls `is_vad()` for each received audio frame before passing to ASR
- **State**: VAD stores per-connection state directly on the `ConnectionHandler` object (`_vad_state`, `_vad_context`, `_vad_opus_decoder`, `client_audio_buffer`, `client_voice_window`, `vad_last_voice_time`, etc.)
- **Cleaning**: `release_conn_resources(conn)` called when connection closes to free VAD state
