# config/

## Responsibility

Central configuration and logging infrastructure for the Xiaozhi server. This directory provides:
- **`config_loader.py`**: Configuration loading, merging (default → custom → API), directory initialization, and async retrieval of per-device private config from the Java manager API.
- **`settings.py`**: Config file validation — ensures `data/.config.yaml` exists and warns if both local and API config are mixed.
- **`logger.py`**: Loguru-based logging setup with configurable console/file output, module-abbreviation strings embedded in log format, and per-connection loggers.
- **`manage_api_client.py`**: Singleton HTTP client (`httpx.AsyncClient`) for communicating with the external Java management API — fetches server config, agent models, correct words, reports chat history, generates chat summaries/titles, and looks up devices in the address book.

## Design

- **Layered configuration merge**: `merge_configs()` recursively merges `config.yaml` (defaults) → `data/.config.yaml` (user overrides) with custom overrides taking priority. If a `manager-api.url` is configured, config is fetched from the remote API instead of local merging.
- **Singleton HTTP client**: `ManageApiClient` uses a `__new__` singleton pattern with per-event-loop `httpx.AsyncClient` instances. Implements exponential-backoff retry (configurable max_retries/retry_delay) for transient network errors.
- **Loguru binding pattern**: Each module gets a `logger.bind(tag=TAG)` for structured logging. Connection-specific loggers are created via `create_connection_logger()` with a module-abbreviation string (e.g., "SiFuChEdGnIn").
- **Async config fetch**: `get_config_from_api_async()` and `get_private_config_from_api()` run asynchronously, the latter fetching agent models and correct words in parallel via `asyncio.gather()`.
- **Business exception types**: `DeviceNotFoundException` and `DeviceBindException` are raised by the API client and caught upstream in `connection.py` to trigger device-binding flows.

## Flow

1. `load_config()` → checks `cache_manager.get(CONFIG, "main_config")` → if miss, reads `config.yaml` + `data/.config.yaml`.
2. If `manager-api.url` exists: calls `get_config_from_api_async()` (POST `/config/server-base`) → merges with local server settings → caches result.
3. If no API: calls `merge_configs(default, custom)` → caches result.
4. `ensure_directories()` creates all required output directories (log, ASR/TTS, model).
5. When a device connects, `get_private_config_from_api()` fetches per-device model selection, correct words, and prompt overrides via parallel `asyncio.gather` calls.
6. `setup_logging()` → reads log config → removes default Loguru handlers → adds stdout handler + rotating file handler (10 MB rotation, 30-day retention).

## Integration

- **Consumed by**: `app.py` (main entrypoint), `core/connection.py` (per-connection private config), `core/websocket_server.py` (shared modules), `core/http_server.py`, all `core/utils/` modules (logging), and all provider modules.
- **Depends on**: `pyyaml` for YAML parsing, `httpx` for async HTTP, `loguru` for logging, `core/utils/cache/` for config caching.
- **External**: Talks to the Java `manager-api` (POST endpoints: `/config/server-base`, `/config/agent-models`, `/config/correct-words`, `/agent/chat-history/report`, `/agent/chat-summary/{id}/save`, `/agent/chat-title/{id}/generate`, `/device/address-book/lookup`).
