# ASR DTO (Data Transfer Objects)

## Responsibility

Defines the `InterfaceType` enum used across all ASR providers to classify whether a provider operates in streaming, non-streaming, or local mode.

## Design

- **`InterfaceType(Enum)`**: Three variants:
  - `STREAM` — WebSocket-based streaming ASR (audio sent frame-by-frame, results returned incrementally)
  - `NON_STREAM` — REST/batch ASR (audio accumulated then sent as one request)
  - `LOCAL` — Locally running ASR model (e.g., FunASR, Vosk, Sherpa-ONNX)

## Integration

- **Used by**: All `ASRProvider` implementations set `self.interface_type` in their `__init__` to signal their mode
- **Used by**: `ASRProviderBase.receive_audio()` checks `interface_type != STREAM` to decide whether to batch audio before calling `handle_voice_stop()`
