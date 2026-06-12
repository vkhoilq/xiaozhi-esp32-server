# core/utils/cache/

## Responsibility

Global in-memory caching subsystem for the Xiaozhi server. Provides a thread-safe, multi-strategy cache used across the entire codebase to reduce redundant API calls, file reads, and computational overhead. Contains:

- **`strategies.py`**: Core data structures — `CacheStrategy` enum (TTL, LRU, FIXED_SIZE, TTL_LRU) and `CacheEntry` dataclass (value, timestamp, ttl, access_count, last_access) with `is_expired()` and `touch()` methods.
- **`config.py`**: `CacheType` enum defining all cache namespaces (LOCATION, WEATHER, LUNAR, INTENT, IP_INFO, CONFIG, DEVICE_PROMPT, VOICEPRINT_HEALTH, AUDIO_DATA) and `CacheConfig` dataclass with preset configurations per type (strategy, TTL, max_size, cleanup_interval).
- **`manager.py`**: `GlobalCacheManager` singleton — thread-safe get/set/delete/clear/invalidate operations with per-cache-type locking, periodic expired-entry cleanup, and LRU reordering for TTL_LRU and LRU strategies.

## Design

- **Singleton pattern**: `cache_manager = GlobalCacheManager()` is created at module level and imported directly by all consumers (e.g., `from core.utils.cache.manager import cache_manager`).
- **Multi-namespace isolation**: Each `CacheType` + optional namespace creates an independent cache dictionary with its own `CacheConfig`, lock, and eviction policy.
- **Per-type strategy selection**: Different data types use different strategies:
  - `TTL` (time-based expiry): LOCATION (no TTL, manual invalidation), IP_INFO (24h), WEATHER (8h), LUNAR (30d), DEVICE_PROMPT (manual), VOICEPRINT_HEALTH (10min), AUDIO_DATA (10min).
  - `TTL_LRU` (TTL + eviction): INTENT (10min, max 1000).
  - `FIXED_SIZE` (size cap only): CONFIG (max 20, no TTL).
- **Thread safety**: Per-cache `threading.RLock` for all read/write operations, plus a `_global_lock` for cache creation. Logging is lazily initialized to avoid circular imports with `config.logger`.
- **LRU implementation**: For LRU/TTL_LRU strategies, caches use `OrderedDict`. `get()` moves accessed entries to the end; eviction removes the first (oldest) entry when `max_size` is exceeded.
- **Periodic cleanup**: `_maybe_cleanup()` runs every `cleanup_interval` (default 60s) to remove expired entries and update stats.

## Flow

```
# Set a value
cache_manager.set(CacheType.WEATHER, "广州", weather_report)
  → _get_cache_name("weather") → _get_or_create_cache()
    → creates OrderedDict/plain dict with CacheConfig for WEATHER type
  → creates CacheEntry(value, timestamp, ttl=28800)
  → stores in dict (LRU: move to end)
  → checks max_size (1000) → evicts oldest if exceeded
  → _maybe_cleanup() → removes expired entries if interval elapsed

# Get a value
cache_manager.get(CacheType.WEATHER, "广州")
  → _get_cache_name("weather")
  → checks if key exists → checks is_expired()
    → if expired: delete and return None
    → if valid: touch() (update last_access, increment access_count)
    → if LRU: move to end
    → return value

# Invalidate
cache_manager.invalidate_pattern(CacheType.DEVICE_PROMPT, "device_123")
  → deletes all keys containing "device_123"

# Clear
cache_manager.clear(CacheType.CONFIG)
  → empties the entire config cache namespace
```

## Integration

- **Consumed by**: `config/config_loader.py` (config caching), `core/utils/util.py` (IP info, audio data caching), `core/utils/prompt_manager.py` (prompt templates, device prompts, location, weather caching), `core/utils/voiceprint_provider.py` (health check caching).
- **Depends on**: Standard library only (`threading`, `time`, `collections.OrderedDict`). Lazily imports `config.logger` to avoid circular dependency.
- **No external dependencies**: Pure in-memory cache. No Redis, Memcached, or disk persistence. Cleanup is timer-based within the `set()` call path (not a background thread).
