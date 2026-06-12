# TTS DTO (Data Transfer Objects)

## Responsibility

Defines the data structures and enums used by the TTS text processing pipeline to communicate between threads and components.

## Design

- **`SentenceType(Enum)`**: Three-stage sentence lifecycle used in audio queue tuples:
  - `FIRST` — first sentence of a response (resets streaming state)
  - `MIDDLE` — continuation sentence (audio data frames)
  - `LAST` — final sentence (triggers report and cleanup)

- **`ContentType(Enum)`**: Types of content that can be synthesized:
  - `TEXT` — text string to be converted to speech
  - `FILE` — pre-existing audio file to play
  - `ACTION` — action command (not currently used)

- **`InterfaceType(Enum)`**: TTS provider interface classification:
  - `DUAL_STREAM` — bidirectional WebSocket (text sent, audio received on same connection)
  - `SINGLE_STREAM` — single HTTP streaming connection (audio streamed as response body)
  - `NON_STREAM` — batch/offline synthesis (returns complete audio before playback)

- **`TTSMessageDTO`**: Value object carrying synthesis tasks between threads:
  - `sentence_id` — links messages to a conversation session
  - `sentence_type` — FIRST/MIDDLE/LAST
  - `content_type` — TEXT/FILE/ACTION
  - `content_detail` — text to synthesize (for TEXT type)
  - `content_file` — audio file path (for FILE type)

## Integration

- **Used by**: `TTSProviderBase.tts_text_queue` — messages are `TTSMessageDTO` instances
- **Used by**: `TTSProviderBase.tts_audio_queue` — audio queue tuples of `(SentenceType, audio_datas, text, sentence_id)`
- **Used by**: All TTS provider implementations import these types to construct their processing logic
