# core/handle/textHandler/

## Responsibility

Concrete `TextMessageHandler` implementations for each WebSocket text message type. Each file handles exactly one message type, providing a clean separation of concerns:

- **`helloMessageHandler.py`** (`HelloTextMessageHandler`): Handles `type: "hello"` — device handshake. Delegates to `helloHandle.handleHelloMessage()` to configure audio params (format, sample rate), detect client features (MCP, emoji), and send the welcome message with session_id.
- **`abortMessageHandler.py`** (`AbortTextMessageHandler`): Handles `type: "abort"` — client interrupt signal. Calls `handleAbortMessage()` to clear queues, stop TTS, and reset speak status.
- **`listenMessageHandler.py`** (`ListenTextMessageHandler`): Handles `type: "listen"` — voice activity state machine. Three states:
  - `start`: Device switches from playback to recording mode — resets audio states.
  - `stop`: Device stops recording — triggers ASR (sends stop request for streaming ASR, or processes buffered audio for non-streaming ASR).
  - `detect`: Device sends recognized text — handles wake words, `[device_call]` directives, and normal text-to-chat flow.
- **`iotMessageHandler.py`** (`IotTextMessageHandler`): Handles `type: "iot"` — device IoT state reports. Two sub-types: `descriptors` (device capability declarations) and `states` (device state updates). Delegates to `handleIotDescriptors` / `handleIotStatus`.
- **`mcpMessageHandler.py`** (`McpTextMessageHandler`): Handles `type: "mcp"` — Model Context Protocol messages from the device. Delegates to `handle_mcp_message()` for MCP JSON-RPC payload processing.
- **`serverMessageHandler.py`** (`ServerTextMessageHandler`): Handles `type: "server"` — server control messages from authorized clients. Two actions: `update_config` (re-fetches config from manager API and re-initializes modules) and `restart` (spawns a new process and exits). Both require secret validation when using API-based config.
- **`pingMessageHandler.py`** (`PingMessageHandler`): Handles `type: "ping"` — optional WebSocket heartbeat. Responds with `{"type": "pong", "timestamp": "..."}`. Controlled by `enable_websocket_ping` config flag.

## Design

- **Single Responsibility Principle**: Each handler class maps 1:1 to a `TextMessageType` enum value. Logic is minimal — handlers delegate to domain functions in sibling modules (`abortHandle`, `helloHandle`, etc.).
- **Abstract base class**: All handlers extend `TextMessageHandler` from the parent `handle/` package, implementing `handle(conn, msg_json)` and `message_type` property.
- **Registry auto-discovery**: `TextMessageHandlerRegistry._register_default_handlers()` instantiates all seven handlers. Adding a new type requires creating a handler class + registering it in the registry.
- **Lazy import pattern**: Uses `TYPE_CHECKING` guard for `ConnectionHandler` type hints to avoid circular imports, since handlers are imported by the registry which is imported by connection.
- **Config validation in server handler**: `ServerTextMessageHandler` validates the incoming secret against `manager-api.secret` before allowing config updates or restarts.

## Flow

```
WebSocket text message arrives
  → ConnectionHandler._route_message()
    → textHandle.handleTextMessage(conn, message)
      → TextMessageProcessor.process_message()
        → json.loads(message) → get "type" field
          → TextMessageHandlerRegistry.get_handler(type)
            → e.g., HelloTextMessageHandler.handle(conn, msg_json)
              → handleHelloMessage(conn, msg_json)  # in helloHandle.py
```

## Integration

- **Consumed by**: `core/handle/textMessageHandlerRegistry.py` imports and registers all handlers. The registry is used by `core/handle/textMessageProcessor.py` which is called from `core/handle/textHandle.py`.
- **Depends on**: `core/handle/` domain modules (`abortHandle`, `helloHandle`, `receiveAudioHandle`), `core/providers/` (ASR interface types, MCP client, IoT handlers), `core/utils/` (dialogue, util), `core/connection.py` (type hints).
- **No external dependencies** — these handlers are pure orchestration layers between the WebSocket protocol and the domain logic in parent handle modules.
