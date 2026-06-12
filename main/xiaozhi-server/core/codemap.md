# core/

## Responsibility

The core runtime engine of the Xiaozhi server. This directory implements the server infrastructure — WebSocket and HTTP listeners, connection lifecycle management, authentication, and message routing. It contains:

- **`websocket_server.py`** (`WebSocketServer`): Listens on a configurable port (default 8000) for ESP32 device WebSocket connections at `/xiaozhi/v1/`. Handles initial module initialization (VAD, ASR, LLM, Memory, Intent shared instances), authentication (token verification + device whitelist), and delegates each connection to a `ConnectionHandler`.
- **`connection.py`** (`ConnectionHandler`): Per-connection state machine — the largest file (~1800 lines). Manages the full lifecycle: device auth, background config fetch, component (re-)initialization, audio VAD/ASR processing, LLM-driven dialogue with function calling/direct_answer, TTS streaming with `AudioRateController`, MCP communication, device binding flow, timeout detection, and graceful teardown with memory saving.
- **`http_server.py`** (`SimpleHttpServer`): Runs an aiohttp HTTP server on port 8003 serving OTA firmware endpoints (`/xiaozhi/ota/`) and Vision analysis endpoints (`/mcp/vision/explain`).
- **`auth.py`** (`AuthManager`, `AuthenticationError`): HMAC-SHA256 token generation and verification for WebSocket connections. Used by both the WebSocket server and the OTA handler.

Subdirectories:
- **`api/`**: HTTP request handlers (OTA, Vision) inheriting from `BaseHandler`.
- **`handle/`**: Message routing — receives parsed JSON messages from `ConnectionHandler` and dispatches to domain-specific handlers (hello, abort, listen, IoT, MCP, server, ping).
- **`providers/`**: Pluggable implementations for VAD, ASR, TTS, LLM, Memory, Intent, VLLM, and tools.
- **`mcp/`**: Model Context Protocol client and tool integrations.
- **`utils/`**: Shared utilities: cache, dialogue, prompt manager, audio processing, text utils, factory functions for all provider types.

## Design

- **Per-connection architecture**: One `ConnectionHandler` instance per WebSocket connection, with its own dialogue state, VAD/ASR/TTS configuration, audio buffers, queues, event loop references, and I/O threads. Shared module instances (Local ASR, VAD) are passed from `WebSocketServer`.
- **Background initialization**: `_background_initialize()` fetches per-device private config from the API asynchronously and then initializes components in a thread pool, never blocking the main WebSocket receive loop.
- **Message routing with Strategy pattern**: Incoming text messages are parsed by `textHandle.py` → `TextMessageProcessor` → `TextMessageHandlerRegistry` dispatches to type-specific handlers (hello, abort, listen, IoT, MCP, server, ping). Audio bytes go through VAD → ASR pipeline.
- **`direct_answer` virtual tool**: A pseudo-function injected into the `function_call` intent flow to prevent small models from hallucinating tool calls. When the LLM selects `direct_answer`, the response argument is streamed directly to TTS without invoking any real tool.
- **Audio rate control**: `AudioRateController` implements frame-accurate 60ms timed audio packet sending with pre-buffer (first 5 packets sent immediately) and dynamic/static delay modes.
- **Device binding flow**: If the manager API returns `DeviceNotFoundException` or `DeviceBindException`, the connection enters a "needs bind" state where all messages are discarded and periodic binding-code prompts are played.
- **Auth chain**: Two auth systems coexist — `AuthManager` (HMAC-SHA256 for WebSocket tokens) and `AuthToken` (JWT + AES-GCM for Vision API tokens, in `core/utils/auth.py`).

## Flow

### WebSocket Connection Lifecycle
1. `WebSocketServer._handle_connection(ws)` → `_handle_auth()` checks whitelist or verifies Bearer token.
2. Creates `ConnectionHandler(config, shared_vad, shared_asr, llm, memory, intent, server)`.
3. `handle_connection(ws)` → reads headers (device-id, client-id, IP) → starts timeout checker → sends welcome message → enters `async for message in websocket:` loop.
4. `_background_initialize()` fires: fetches per-device config from API → re-initializes components if needed.
5. Message routing: text → `handleTextMessage()` → `TextMessageProcessor` → handler; bytes → VAD → ASR → `startToChat()`.
6. On disconnect: `_save_and_close()` spawns daemon threads to save memory + generate chat title, then closes WebSocket.

### Text Message Routing
- `handleTextMessage()` → `message_processor.process_message()` → JSON parse → get handler by `type` field → `handler.handle(conn, msg_json)`.
- Supported types: `hello` (handshake + audio params), `abort` (client interrupt), `listen` (VAD state control + ASR trigger), `iot` (device descriptors/states), `mcp` (MCP protocol payloads), `server` (remote config update/restart), `ping` (heartbeat).

### LLM Chat Flow
1. Audio → VAD detects voice → ASR transcribes → `startToChat()` with recognized text.
2. `handle_user_intent()` checks exit commands → wake words → intent analysis.
3. If `function_call` intent: `ConnectionHandler.chat()` builds dialogue → LLM `response_with_functions()` streams tokens → detects `direct_answer` or real tool calls → `UnifiedToolHandler.handle_llm_function_call()` executes tools → results fed back into dialogue → recursive LLM call (max depth 5).
4. TTS output: text chunks → `tts.text_queue` → `tts_one_sentence()` → Opus encoding → `AudioRateController` → WebSocket send.

## Integration

- **Consumed by**: `app.py` launches `WebSocketServer` and `SimpleHttpServer`.
- **Depends on**: `config/` (settings, logger, manage API client), all `core/providers/` (VAD, ASR, TTS, LLM, Memory, Intent), `core/utils/` (cache, dialogue, prompt manager, audio tools), `plugins_func/` (function plugins).
- **External services**: Manager API (per-device config), MCP servers (MCP protocol), external TTS/ASR/LLM APIs, voiceprint servers, weather/news APIs.
- **Subdirectory dependency**: `core/handle/` ← `core/connection.py`; `core/api/` ← `core/http_server.py`; `core/utils/` ← all core modules.
