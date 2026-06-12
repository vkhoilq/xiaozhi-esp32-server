# Plugins Function System (`plugins_func/`)

## Responsibility
Provides the **decorator-based plugin registration framework** and **dynamic module loader** for the xiaozhi-esp32-server. This is the lower-level infrastructure that individual plugin functions use to declare themselves as callable tools. It defines the core types (`Action`, `ActionResponse`, `FunctionItem`, `ToolType`), the global registry, and the dynamic import mechanism that auto-discovers plugins.

## Design
### `register.py` — Core Registry Framework
- **Decorator-Based Registration**: `@register_function(name, desc, type)` is the primary API for plugin authors. It creates a `FunctionItem` and inserts it into the global `all_function_registry` dictionary.
- **Action Enum Lifecycle**: `Action` enum defines the post-execution behavior:
  - `ERROR` (-1): Tool execution failed
  - `NOTFOUND` (0): Tool not found
  - `NONE` (1): No further action needed
  - `RESPONSE` (2): Directly respond with the result (no LLM re-query)
  - `REQLLM` (3): Pass result back to LLM for response generation
  - `RECORD` (4): Record tool call in dialogue history without LLM call
- **ToolType (Legacy)**: A separate `ToolType` enum with `code`-based dispatch:
  - `NONE` (1): No-op after call
  - `WAIT` (2): Call and wait for return
  - `CHANGE_SYS_PROMPT` (3): Role switch — needs `conn` argument
  - `SYSTEM_CTL` (4): System control — needs `conn` argument
  - `IOT_CTL` (5): IoT control — needs `conn` argument
  - `MCP_CLIENT` (6): MCP client operations
- **FunctionItem**: Data class bundling `name`, `description` (OpenAI format dict), `func` (callable), and `type` (ToolType).
- **DeviceTypeRegistry**: Maps device capability signatures to their available functions, supporting IoT device type identification.
- **FunctionRegistry**: A class-based registry wrapper with registration, unregistration, and description listing methods.

### `loadplugins.py` — Dynamic Module Loader
- **`auto_import_modules(package_name)`**: Uses `importlib` and `pkgutil.iter_modules` to discover and import all modules within a package. This triggers the `@register_function` decorators, populating `all_function_registry` at import time.

## Flow
1. **Plugin Loading**: `UnifiedToolHandler._initialize()` calls `auto_import_modules("plugins_func.functions")`.
2. **Module Discovery**: `loadplugins.py` iterates all `.py` files in the `plugins_func/functions/` package and imports each one.
3. **Decorator Execution**: Each imported module's `@register_function` decorator runs at import time, creating `FunctionItem` objects and storing them in the global `all_function_registry`.
4. **Tool Exposure**: `ServerPluginExecutor.get_tools()` queries `all_function_registry` for configured function names, wraps them into `ToolDefinition` objects.
5. **Function Invocation**: When LLM calls a function, `ServerPluginExecutor.execute()` looks up the function in `all_function_registry`, dispatches based on `ToolType.code`, and returns the `ActionResponse`.

## Integration
- **Dependencies**: `config.logger`, `enum`, standard library only
- **Consumers**:
  - `core/providers/tools/server_plugins/plugin_executor.py` — reads `all_function_registry`
  - `core/providers/tools/base/tool_types.py` — references `Action` enum for `ToolDefinition`
  - `core/providers/tools/unified_tool_handler.py` — calls `auto_import_modules()`
  - All plugin function files in `plugins_func/functions/` — use `@register_function`
- **Global State**: `all_function_registry` is a module-level dictionary, making it a singleton registry accessible throughout the application.
