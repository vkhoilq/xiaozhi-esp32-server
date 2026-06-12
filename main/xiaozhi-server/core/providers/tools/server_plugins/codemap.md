# Server Plugins (`core/providers/tools/server_plugins/`)

## Responsibility
Bridges the **decorator-based plugin system** (`plugins_func`) into the unified tool framework. The `ServerPluginExecutor` wraps plugin functions registered via `@register_function` into the `ToolExecutor` interface, enabling LLM function-calling to invoke plugin functions with proper argument passing. It also handles tool description customization per-connection (e.g., news source descriptions).

## Design
- **Adapter Pattern**: `ServerPluginExecutor` adapts the global `all_function_registry` dictionary (from `plugins_func.register`) to the `ToolExecutor` ABC interface. Plugin functions with their `FunctionItem` metadata are wrapped into `ToolDefinition` objects.
- **Function Type Dispatch**: The executor checks `func_item.type.code` to determine how to invoke:
  - `code 4, 5` (SYSTEM_CTL, IOT_CTL): Passes `conn` as first argument
  - `code 2` (WAIT): No `conn` argument
  - `code 3` (CHANGE_SYS_PROMPT): Passes `conn` as first argument
  - Default: No `conn` argument
- **Per-Connection Configuration**: Tool descriptions are sourced from `conn.config["plugins"]`, allowing each connection to customize the LLM-facing description (e.g., news sources). The `_init_news_source_description()` method dynamically updates the news tool's parameter description based on the connection's plugin config.
- **Selective Registration**: Only functions listed in `necessary_functions` (hardcoded: `handle_exit_intent`, `get_lunar`) plus the connection's configured `functions` list are exposed as tools. This provides per-intent-model control over which plugins are available.

## Flow
1. **`get_tools()`**:
   - Reads `necessary_functions` list + connection's `config["Intent"][...]["functions"]` list
   - Merges into `all_required_functions` (deduplicated)
   - For each function name:
     - Looks up `all_function_registry[name]`
     - If found, applies plugin-specific description overrides from `conn.config.get("plugins", {})`
     - For `get_news_from_newsnow`: calls `_init_news_source_description()` to update source parameter
     - Wraps into `ToolDefinition` with `ToolType.SERVER_PLUGIN`
2. **`execute(conn, tool_name, arguments)`**:
   - Looks up `all_function_registry[tool_name]`
   - Determines invocation style based on `func_item.type.code`
   - Calls the function with appropriate arguments
   - Returns the `ActionResponse` from the plugin function

## Integration
- **Dependencies**:
  - `plugins_func.register.all_function_registry`, `Action`, `ActionResponse`
  - `plugins_func.register.ToolType` (the one in register.py with `code`-based dispatch)
  - `..base.ToolType` (enum), `ToolDefinition`, `ToolExecutor`
- **Consumers**: `UnifiedToolHandler` creates `ServerPluginExecutor` and registers it for `ToolType.SERVER_PLUGIN`.
- **Configuration**: Functions enabled via `config["Intent"][...]["functions"]` list; plugin-specific descriptions via `config["plugins"][function_name]["description"]`.
