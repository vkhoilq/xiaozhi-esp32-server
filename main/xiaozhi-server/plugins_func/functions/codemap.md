# Plugin Functions Directory (`plugins_func/functions/`)

## Responsibility
Contains **all built-in plugin function implementations** that the LLM can invoke via function-calling. Each file defines one or more functions decorated with `@register_function`, which are auto-discovered at startup by `auto_import_modules()`. These plugins extend the assistant's capabilities with web search, news, weather, smart home control, music playback, calendar/lunar info, role switching, exit handling, device calling, and knowledge base retrieval.

## Design
- **Self-Registering Modules**: Each file uses `@register_function(name, desc, tool_type)` at module level. Importing the module triggers registration into the global `all_function_registry`.
- **Conn-Aware Execution**: Functions with `ToolType.SYSTEM_CTL` (code 4/5) receive the connection handler (`conn`) as their first parameter, giving them access to WebSocket, config, device state, and other runtime context.
- **ActionResponse Convention**: All plugin functions return `ActionResponse(action, result, response)`. The `action` field determines how the system processes the result:
  - `Action.REQLLM`: Pass result to LLM for response generation (most common)
  - `Action.RESPONSE`: Directly use `response` text (no LLM re-query)
  - `Action.NONE`: Silent execution
  - `Action.ERROR`: Error handling
  - `Action.RECORD`: Record the call without triggering LLM
- **Caching Strategy**: Several plugins (`get_weather`, `get_lunar`, `call_device` address-book) use `cache_manager` with TTL-based caching to avoid redundant API calls.

## Plugin Summary

### `web_search.py` — Internet Search
- **Function**: `web_search(conn, query)`
- **ToolType**: `SYSTEM_CTL`
- **Providers**: "metaso" (秘塔搜索 API) or "tavily" (Tavily Search API)
- **Config**: `plugins.web_search.provider`, `api_key`, `max_results`
- **Flow**: Calls the configured search API → formats results as `【联网搜索结果】` → returns `Action.REQLLM` with result text.

### `get_weather.py` — Weather Query
- **Function**: `get_weather(conn, location, lang)`
- **ToolType**: `SYSTEM_CTL`
- **Provider**: QWeather API (和风天气)
- **Features**: IP-based location auto-detection, 7-day forecast, caching via `CacheType.WEATHER`
- **Config**: `plugins.get_weather.api_key`, `api_host`, `default_location`

### `get_news_from_newsnow.py` — News (NewsNow API)
- **Function**: `get_news_from_newsnow(conn, source, detail, lang)`
- **ToolType**: `SYSTEM_CTL`
- **Source**: [newsnow.busiyi.world](https://newsnow.busiyi.world) with 40+ channels (V2EX, 知乎, 微博, etc.)
- **Features**: Random news selection, detail fetching via MarkItDown HTML cleaning, per-connection news source configuration

### `get_news_from_chinanews.py` — News (ChinaNews RSS)
- **Function**: `get_news_from_chinanews(conn, category, detail, lang)`
- **ToolType**: `SYSTEM_CTL`
- **Source**: ChinaNews.com RSS feeds (社会/国际/财经 categories)
- **Features**: RSS XML parsing, detail fetching via BeautifulSoup, category mapping

### `hass_get_state.py` — Home Assistant State Query
- **Function**: `hass_get_state(conn, entity_id)`
- **ToolType**: `SYSTEM_CTL`
- **Flow**: Queries Home Assistant REST API → returns device state including media title, volume, brightness, color, color temperature
- **Config**: `plugins.home_assistant` (or legacy `plugins.hass_get_state`) with `base_url`, `api_key`, `devices`

### `hass_set_state.py` — Home Assistant State Control
- **Function**: `hass_set_state(conn, entity_id, state)`
- **ToolType**: `SYSTEM_CTL`
- **Actions**: turn_on/off, brightness up/down/value, set_color, set_kelvin, volume up/down/set/mute, pause/continue — with domain-specific logic (cover, vacuum, media_player)

### `hass_play_music.py` — Home Assistant Music Playback
- **Function**: `hass_play_music(conn, entity_id, media_content_id)`
- **ToolType**: `SYSTEM_CTL`
- **Flow**: Calls Music Assistant service via HA REST API → returns playback confirmation
- **Note**: Runs via `asyncio.run_coroutine_threadsafe` to bridge sync function to async HA call

### `hass_init.py` — Home Assistant Initialization (Helper)
- **Functions**: `append_devices_to_prompt(conn)`, `initialize_hass_handler(conn)`
- **Not directly registered** as a tool — provides shared helper functions for HA plugins
- **`append_devices_to_prompt`**: Injects device list into the system prompt for LLM context
- **`initialize_hass_handler`**: Resolves HA config from either `home_assistant` or legacy `hass_get_state` config key

### `play_music.py` — Local Music Playback
- **Function**: `play_music(conn, song_name)`
- **ToolType**: `SYSTEM_CTL`
- **Flow**: Schedules async `handle_music_command` via `conn.loop.create_task` → scans local music directory → fuzzy matches song name (`difflib.SequenceMatcher`) → queues TTS messages with audio file
- **Config**: `plugins.play_music.music_dir`, `music_ext`, `refresh_time`
- **Output**: Pushes `TTSMessageDTO` entries to `conn.tts.tts_text_queue` with `ContentType.FILE` for the audio stream

### `get_time.py` / `get_lunar` — Lunar Calendar
- **Function**: `get_lunar(date=None, query=None)`
- **ToolType**: `WAIT` (no conn needed)
- **Library**: `cnlunar` — computes lunar date, 干支, zodiac, 八字, 节气, 宜忌, 星座, 纳音, etc.
- **Caching**: Via `CacheType.LUNAR`

### `change_role.py` — Role Switching
- **Function**: `change_role(conn, role, role_name)`
- **ToolType**: `CHANGE_SYS_PROMPT` (triggers system prompt replacement)
- **Roles**: "英语老师", "机车女友", "好奇小男孩" — each with a detailed character prompt template
- **Flow**: Selects prompt template → replaces `{{assistant_name}}` → calls `conn.change_system_prompt()` → returns success response

### `handle_exit_intent.py` — Exit/Chat End
- **Function**: `handle_exit_intent(conn, say_goodbye)`
- **ToolType**: `SYSTEM_CTL`
- **Flow**: Sets `conn.close_after_chat = True` → returns goodbye message
- **Essential always-registered function**: Always included in tool list (part of `necessary_functions`)

### `call_device.py` — Device-to-Device Calling
- **Function**: `call_device(conn, nickname)`
- **ToolType**: `SYSTEM_CTL`
- **Flow**: Queries Java management API for address book → validates permissions → forwards call request → handles offline/online status
- **Config**: `manager-api.url`, `manager-api.secret`

### `search_from_ragflow.py` — RAG Knowledge Base
- **Function**: `search_from_ragflow(conn, question)`
- **ToolType**: `SYSTEM_CTL`
- **Flow**: POST to RAGFlow retrieval API → extracts up to 5 relevant chunks → returns as `【关于问题...查到知识库如下】`
- **Config**: `plugins.search_from_ragflow.base_url`, `api_key`, `dataset_ids`

## Integration
- **Dependencies**: Various (requests, bs4, cnlunar, markitdown, etc.), `plugins_func.register` (all files), `core.utils.cache.manager` (weather, lunar), `core.handle.sendAudioHandle` (play_music)
- **Discovery**: Auto-imported by `loadplugins.auto_import_modules("plugins_func.functions")` during `UnifiedToolHandler._initialize()`
- **Consumers**: `ServerPluginExecutor` reads `all_function_registry` and exposes configured functions as tools for LLM function-calling
