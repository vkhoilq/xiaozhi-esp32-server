# Tools Providers Directory (`core/providers/tools/`)

## Responsibility
Implements the **unified tool/function-calling subsystem** for the xiaozhi-esp32-server. This directory provides the central orchestration layer for tool registration, discovery, and execution, supporting five distinct tool origins: server-side plugins, server-side MCP (Model Context Protocol), device-side IoT, device-side MCP, and MCP endpoints. It bridges LLM function-call requests to concrete executable actions.

## Design
- **Strategy + Registry Pattern**: `ToolManager` acts as a central registry that maps `ToolType` enum values to `ToolExecutor` instances. Each executor implements the `ToolExecutor` ABC with `execute()`, `get_tools()`, and `has_tool()` methods.
- **Facade Pattern**: `UnifiedToolHandler` provides a simplified interface over the entire tool subsystem — initialization, function listing, execution, IoT tool registration, and cleanup — hiding the complexity of multiple executor types from callers.
- **Plugin Architecture via Auto-Import**: During `_initialize()`, the handler calls `auto_import_modules("plugins_func.functions")` to dynamically discover and register all `@register_function`-decorated plugin functions.
- **Type-Safe Tool Definitions**: `ToolType` enum (`SERVER_PLUGIN`, `SERVER_MCP`, `DEVICE_IOT`, `DEVICE_MCP`, `MCP_ENDPOINT`) and `ToolDefinition` dataclass provide structured metadata for tool registration.
- **Caching**: `ToolManager` caches tool definitions and function descriptions, invalidating on any registration change via `_invalidate_cache()`.

## Flow
1. **Initialization** (`UnifiedToolHandler.__init__`):
   - Creates `ToolManager` instance
   - Instantiates all 5 executors: `ServerPluginExecutor`, `ServerMCPExecutor`, `DeviceIoTExecutor`, `DeviceMCPExecutor`, `MCPEndpointExecutor`
   - Registers each executor in `ToolManager` with its corresponding `ToolType`
2. **Async Initialization** (`_initialize()`):
   - Auto-imports plugin function modules
   - Initializes server MCP connections
   - Connects to MCP endpoint (if configured)
   - Initializes Home Assistant prompt (if configured)
   - Calls `current_support_functions()` to log available tools
3. **Tool Execution** (`handle_llm_function_call`):
   - Receives LLM function call data (single or multi-call)
   - Sends display message to device about the tool being invoked
   - Delegates to `ToolManager.execute_tool(tool_name, arguments)` which:
     - Looks up the tool's type → finds the registered executor → calls `executor.execute()`
4. **Tool Discovery** (`get_functions()`): Returns OpenAI-format function descriptions for LLM context construction.
5. **IoT Registration** (`register_iot_tools`): Called when device sends IoT descriptors — registers execute/query tools dynamically.
6. **Cleanup** (`cleanup()`): Closes server MCP connections and MCP endpoint WebSocket.

## Integration
- **Dependencies**:
  - `plugins_func.loadplugins.auto_import_modules` — dynamic plugin discovery
  - `plugins_func.register` — `Action`, `ActionResponse`, `all_function_registry`
  - `core.handle.sendAudioHandle.send_display_message` — device display notifications
  - Sub-executors: `server_plugins`, `server_mcp`, `device_iot`, `device_mcp`, `mcp_endpoint`
- **Consumers**: The core connection handler (`core.connection`) creates a `UnifiedToolHandler` and calls `handle_llm_function_call` when the LLM returns function_calls.
- **Configuration**: Intent config selects which functions to enable; MCP servers configured via `data/.mcp_server_settings.json`; MCP endpoint configured via `config.yaml` `mcp_endpoint` field.
