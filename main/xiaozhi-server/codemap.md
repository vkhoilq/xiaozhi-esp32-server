# Repository Atlas: xiaozhi-esp32-server

## Project Responsibility

A Python asyncio WebSocket server for the Xiaozhi ESP32 AI voice assistant device. Implements a full voice pipeline: ASR (speech-to-text) → LLM (large language model) → TTS (text-to-speech) with streaming audio, function calling, IoT device control, and MCP (Model Context Protocol) integration. Supports 15+ ASR engines, 18+ TTS engines, 10+ LLM providers, and multiple memory/intent/vision modules — all swappable via YAML configuration.

## System Entry Points

- **`app.py`**: Async entrypoint — starts WebSocket server (port 8000) and HTTP server (port 8003), manages graceful shutdown.
- **`config.yaml`**: Master configuration defining all modules, API keys, networking, and behavior.
- **`agent-base-prompt.txt`**: Jinja2 system prompt template injected into all LLM conversations.
- **`mcp_server_settings.json`**: MCP server registry for tool integration.

## Architecture Overview

```
ESP32 Device
    │
    ├── WebSocket ──► core/websocket_server.py ──► core/connection.py
    │                                                    │
    │                                    ┌───────────────┼───────────────┐
    │                                    ▼               ▼               ▼
    │                              core/handle/    core/providers/   core/utils/
    │                              (message        (ASR/TTS/LLM/     (factories,
    │                               routing)        VAD/Memory/       dialogue,
    │                                              Intent/Tools)      prompt, cache)
    │
    └── HTTP ──────► core/http_server.py ──► core/api/
                     (OTA + Vision)            (OTA + Vision handlers)
```

## Directory Map (Aggregated)

| Directory | Responsibility Summary | Detailed Map |
|-----------|----------------------|--------------|
| `config/` | Configuration loading, YAML merge, logging setup, manager API client | [View Map](config/codemap.md) |
| `core/` | Per-connection runtime engine: WebSocket/HTTP servers, connection lifecycle, auth, message routing | [View Map](core/codemap.md) |
| `core/api/` | HTTP API handlers: OTA firmware updates, Vision analysis (JWT+AES-GCM auth) | [View Map](core/api/codemap.md) |
| `core/handle/` | Message routing: text dispatch (Strategy+Registry), audio processing, intent analysis, reporting | [View Map](core/handle/codemap.md) |
| `core/handle/textHandler/` | 7 single-responsibility text message handlers (hello, abort, listen, iot, mcp, server, ping) | [View Map](core/handle/textHandler/codemap.md) |
| `core/providers/` | Pluggable provider system: factory-based module initialization for all AI/voice services | [View Map](core/providers/codemap.md) |
| `core/providers/asr/` | 14 ASR adapters (FunASR, Doubao, Aliyun, Baidu, Tencent, OpenAI, Groq, Vosk, Sherpa, Xunfei, Qwen3) | [View Map](core/providers/asr/codemap.md) |
| `core/providers/asr/dto/` | ASR interface type enum (STREAM, NON_STREAM, LOCAL) | [View Map](core/providers/asr/dto/codemap.md) |
| `core/providers/tts/` | 19 TTS adapters (Edge, Doubao, CosyVoice, FishSpeech, GPT-SoVITS, Minimax, Aliyun, Tencent, etc.) | [View Map](core/providers/tts/codemap.md) |
| `core/providers/tts/dto/` | TTS data transfer objects: SentenceType, ContentType, InterfaceType enums | [View Map](core/providers/tts/dto/codemap.md) |
| `core/providers/vad/` | Voice Activity Detection: Silero VAD with dual-threshold hysteresis | [View Map](core/providers/vad/codemap.md) |
| `core/providers/vllm/` | Vision-Language Model adapters (OpenAI-compatible multimodal) | [View Map](core/providers/vllm/codemap.md) |
| `core/providers/intent/` | Intent recognition: base class + 3 implementations (nointent, function_call, intent_llm) | [View Map](core/providers/intent/codemap.md) |
| `core/providers/intent/function_call/` | Function call intent: passthrough to LLM with direct_answer virtual tool | [View Map](core/providers/intent/function_call/codemap.md) |
| `core/providers/intent/intent_llm/` | LLM-based intent: separate LLM call for intent classification with dynamic prompts | [View Map](core/providers/intent/intent_llm/codemap.md) |
| `core/providers/intent/nointent/` | No-op intent: passthrough without intent analysis | [View Map](core/providers/intent/nointent/codemap.md) |
| `core/providers/llm/` | LLM provider base class and system prompt factory for function calling | [View Map](core/providers/llm/codemap.md) |
| `core/providers/llm/AliBL/` | Alibaba Cloud (DashScope) LLM with memory/prompt control | [View Map](core/providers/llm/AliBL/codemap.md) |
| `core/providers/llm/coze/` | Coze (ByteDance) LLM with session-to-conversation mapping | [View Map](core/providers/llm/coze/codemap.md) |
| `core/providers/llm/dify/` | Dify LLM with 3 operational modes (workflow, chat, completion) | [View Map](core/providers/llm/dify/codemap.md) |
| `core/providers/llm/fastgpt/` | FastGPT LLM with think-tag filtering | [View Map](core/providers/llm/fastgpt/codemap.md) |
| `core/providers/llm/gemini/` | Google Gemini with native FunctionDeclaration tool calling | [View Map](core/providers/llm/gemini/codemap.md) |
| `core/providers/llm/homeassistant/` | Home Assistant LLM (non-streaming REST API) | [View Map](core/providers/llm/homeassistant/codemap.md) |
| `core/providers/llm/ollama/` | Ollama local LLM with Qwen3 /no_think directive | [View Map](core/providers/llm/ollama/codemap.md) |
| `core/providers/llm/openai/` | OpenAI-compatible LLM with thinking-mode suppression | [View Map](core/providers/llm/openai/codemap.md) |
| `core/providers/llm/xinference/` | Xinference local LLM with per-chunk think-tag stripping | [View Map](core/providers/llm/xinference/codemap.md) |
| `core/providers/memory/` | Memory provider base class (ABC with set_memory/get_memory) | [View Map](core/providers/memory/codemap.md) |
| `core/providers/memory/mem0ai/` | Mem0 cloud API memory wrapper | [View Map](core/providers/memory/mem0ai/codemap.md) |
| `core/providers/memory/mem_local_short/` | Local LLM-driven summarization with YAML persistence | [View Map](core/providers/memory/mem_local_short/codemap.md) |
| `core/providers/memory/mem_report_only/` | Report-only null object (no summarization) | [View Map](core/providers/memory/mem_report_only/codemap.md) |
| `core/providers/memory/nomem/` | Complete no-op memory provider | [View Map](core/providers/memory/nomem/codemap.md) |
| `core/providers/memory/powermem/` | PowerMem (OceanBase) with dual UserMemory/AsyncMemory modes | [View Map](core/providers/memory/powermem/codemap.md) |
| `core/providers/tools/` | Unified tool orchestration: facade + registry with 5 executor types | [View Map](core/providers/tools/codemap.md) |
| `core/providers/tools/base/` | Tool type system: ToolType, ToolDefinition, ToolExecutor ABC | [View Map](core/providers/tools/base/codemap.md) |
| `core/providers/tools/device_iot/` | Dynamic IoT tool registration from device descriptors | [View Map](core/providers/tools/device_iot/codemap.md) |
| `core/providers/tools/device_mcp/` | Device-side MCP over JSON-RPC/WebSocket | [View Map](core/providers/tools/device_mcp/codemap.md) |
| `core/providers/tools/mcp_endpoint/` | Remote MCP endpoint via independent WebSocket | [View Map](core/providers/tools/mcp_endpoint/codemap.md) |
| `core/providers/tools/server_mcp/` | Server-side MCP with stdio/SSE/HTTP transports and retry | [View Map](core/providers/tools/server_mcp/codemap.md) |
| `core/providers/tools/server_plugins/` | Adapter bridging @register_function plugins to tool system | [View Map](core/providers/tools/server_plugins/codemap.md) |
| `core/utils/` | 7 provider factories, dialogue manager, prompt renderer, audio utils, auth, cache | [View Map](core/utils/codemap.md) |
| `core/utils/cache/` | Singleton GlobalCacheManager with TTL/LRU/FIXED_SIZE/TTL_LRU strategies | [View Map](core/utils/cache/codemap.md) |
| `plugins_func/` | Decorator-based plugin registration framework and dynamic loader | [View Map](plugins_func/codemap.md) |
| `plugins_func/functions/` | 13 built-in plugin functions (weather, news, music, HA, RAG, web search, etc.) | [View Map](plugins_func/functions/codemap.md) |

## Key Data Flows

### Voice Pipeline (Primary)
```
ESP32 → WebSocket → VAD → ASR → Intent Analysis → LLM (with function calling)
    → Tool Execution → TTS → Opus Encoding → AudioRateController → WebSocket → ESP32
```

### Configuration Pipeline
```
config.yaml → data/.config.yaml (overrides) → manager-api (remote) → merged config
    → initialize_modules() → factory creates for each selected_module
```

### Tool Calling Flow
```
LLM response with function_call → UnifiedToolHandler
    → route by ToolType: SERVER_PLUGIN | DEVICE_IOT | DEVICE_MCP | SERVER_MCP | MCP_ENDPOINT
    → execute → ActionResponse → feed back to LLM or respond directly
```