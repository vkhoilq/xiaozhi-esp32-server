# core/api/

## Responsibility

HTTP API handlers for the Xiaozhi server's auxiliary HTTP endpoints (served by `SimpleHttpServer` on port 8003). Contains:

- **`base_handler.py`** (`BaseHandler`): Abstract base class for all HTTP handlers. Provides CORS header injection (`Access-Control-Allow-*`) and OPTIONS request handling for cross-origin browser access.
- **`ota_handler.py`** (`OTAHandler`): Over-The-Air firmware update endpoint (`/xiaozhi/ota/`). Responds to ESP32 devices with connection configuration (WebSocket URL or MQTT broker settings) and optional firmware download URLs. Includes firmware version comparison (`_is_higher_version`) and cached firmware file discovery in `data/bin/`.
- **`vision_handler.py`** (`VisionHandler`): MCP Vision analysis endpoint (`/mcp/vision/explain`). Accepts multipart POST requests with a question + image, verifies JWT+ AES-GCM tokens, initializes the configured VLLM provider, and returns the visual analysis result.

## Design

- **Inheritance pattern**: Both `OTAHandler` and `VisionHandler` extend `BaseHandler`, inheriting CORS support (`_add_cors_headers`, `handle_options`).
- **Token-based auth**: `VisionHandler` uses `AuthToken` (JWT outer + AES-GCM encrypted inner payload) with device ID binding. `OTAHandler` uses `AuthManager` (HMAC-SHA256) for WebSocket token generation.
- **Firmware discovery**: `OTAHandler` maintains a TTL-cached inventory of `data/bin/*.bin` firmware files. Filenames follow the `{model}_{version}.bin` pattern. Version comparison uses numeric tuple parsing (`_parse_version`) — not semver, but handles dotted numeric strings.
- **MQTT gateway fallback**: If `server.mqtt_gateway` is configured, OTA returns MQTT connection parameters (client ID, username, password, pub/sub topics) instead of WebSocket URL.
- **Two-phase Vision auth**: (1) JWT signature verification, (2) encrypted payload decryption + device-id match. Allows a `web_test_client` bypass for testing.

## Flow

### OTA Flow
1. ESP32 sends `POST /xiaozhi/ota/` with device-id/client-id headers + optional JSON body (board info, app version).
2. `OTAHandler.handle_post()` → reads headers for device-model, firmware-version → determines transport (MQTT or WebSocket).
3. If MQTT: builds client ID from group/mac, generates HMAC password.
4. If WebSocket: checks auth_enabled → generates token via `AuthManager.generate_token()`.
5. Refreshes firmware cache (`_refresh_bin_cache_if_needed()`) → looks for higher version for device model → if found, sets `firmware.url` to download endpoint.
6. Returns JSON with `server_time`, transport config (`mqtt` or `websocket`), and optional `firmware` update info.
7. `GET /xiaozhi/ota/` returns the WebSocket URL as plain text for debugging.
8. `GET /xiaozhi/ota/download/{filename}` serves `.bin` files with path traversal protection.

### Vision Flow
1. Client sends `POST /mcp/vision/explain` with multipart form: `question` (text) + image file.
2. `VisionHandler.handle_post()` → verifies Bearer token via `_verify_auth_token()` → checks device-id matches token.
3. Reads image data, validates size (< 5 MB) and format (JPEG/PNG/GIF/BMP/TIFF/WEBP via magic bytes).
4. If using manager-api: fetches per-device private config to get VLLM model selection.
5. Creates VLLM provider instance via `create_instance(type, config)` → calls `vllm.response(question, image_base64)`.
6. Returns JSON `{"success": true, "action": "RESPONSE", "response": "..."}`.

## Integration

- **Consumed by**: `core/http_server.py` registers routes for both handlers. ESP32 devices call OTA endpoints; MCP clients call Vision endpoints.
- **Depends on**: `core/auth.py` (AuthManager for OTA token), `core/utils/auth.py` (AuthToken for Vision JWT), `core/utils/vllm.py` (VLLM factory), `core/utils/util.py` (get_local_ip, get_vision_url, is_valid_image_file), `config/` (logger, config_loader for private config).
- **External**: OTA handler serves firmware files from `data/bin/`. Vision handler talks to external VLLM API (e.g., ChatGLM, Qwen).
