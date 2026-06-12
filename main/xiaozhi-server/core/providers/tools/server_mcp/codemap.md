# Server MCP Tools (`core/providers/tools/server_mcp/`)

## Responsibility
Provides a **server-side MCP (Model Context Protocol) client and manager** that connects to external MCP-compatible services (running as subprocesses via stdio or as remote HTTP/SSE services). The `ServerMCPManager` loads MCP server configurations from `data/.mcp_server_settings.json`, establishes connections, discovers tools, and provides a unified execution interface with automatic reconnection and retry logic.

## Design
- **Multi-Client Manager Pattern**: `ServerMCPManager` manages a dictionary of `ServerMCPClient` instances, each representing a connection to one MCP server. Supports both local subprocess (stdio) and remote (SSE, Streamable HTTP) servers.
- **Protocol Abstraction via `mcp` SDK**: Uses the official `mcp` Python package (`mcp.ClientSession`, `StdioServerParameters`, `stdio_client`, `sse_client`, `streamablehttp_client`) which abstracts the underlying transport protocol.
- **Transport Selection**:
  - **Stdio transport**: When config has `command` (with optional `args` and `env`)
  - **SSE transport**: When config has `url` and `transport` is `sse` (default)
  - **Streamable HTTP transport**: When config has `url` and `transport` is `streamable-http` or `http`
- **Asynchronous Worker Model**: Each `ServerMCPClient` runs a dedicated background worker task (`_worker`) that manages the connection lifecycle, lists tools, and hangs until shutdown.
- **Retry with Reconnection**: `execute_tool()` implements a 3-retry loop with full reconnection of the MCP client between retries, ensuring transient failures are handled gracefully.
- **Tool Name Sanitization**: Uses `sanitize_tool_name()` to convert MCP tool names to safe identifiers, with `name_mapping` for reverse lookup.

## Flow
1. **Server Initialization** (`ServerMCPManager.initialize_servers()`):
   - Loads `data/.mcp_server_settings.json` → reads `mcpServers` map
   - For each server config with `command` or `url`:
     - Creates `ServerMCPClient(srv_config)`
     - Calls `client.initialize()` with a 10s timeout
     - Client spawns `_worker` async task:
       - Establishes transport (stdio/SSE/HTTP) via `AsyncExitStack`
       - Creates `mcp.ClientSession`, calls `session.initialize()`
       - Lists tools via `session.list_tools()`
       - Sanitizes tool names, populates `tools_dict` and `name_mapping`
       - Signals readiness via `_ready_evt`
       - Hangs on `_shutdown_evt.wait()`
     - On success: adds client to `self.clients`, extends `self.tools` list
     - Refreshes `ToolManager` cache after all servers initialized
2. **Tool Execution** (`ServerMCPManager.execute_tool(tool_name, arguments)`):
   - Iterates all clients to find the one containing the tool
   - Calls `client.call_tool(tool_name, arguments)`:
     - Looks up real name from `name_mapping`
     - Calls `session.call_tool(real_name, arguments=arguments)`
     - Thread-safe: if caller loop differs from worker loop, uses `run_coroutine_threadsafe` + `wrap_future`
   - On failure: retries up to 3 times with 2s interval, reconnecting the client on each retry
3. **Cleanup** (`cleanup_all()`):
   - Sets `_shutdown_evt` on each client
   - Awaits worker task completion with 20s timeout
   - Clears clients dictionary

## Integration
- **Dependencies**: `mcp` (PyPI package), `core.utils.util.sanitize_tool_name`, `config.config_loader.get_project_dir`
- **Configuration File**: `data/.mcp_server_settings.json` with structure:
  ```json
  {
    "mcpServers": {
      "server-name": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-something"],
        "env": { "API_KEY": "..." }
      },
      "remote-server": {
        "url": "https://example.com/mcp",
        "transport": "sse",
        "headers": { "Authorization": "Bearer ..." },
        "timeout": 30
      }
    }
  }
  ```
- **Consumers**: `UnifiedToolHandler` creates `ServerMCPExecutor` which creates and manages `ServerMCPManager`. Executor bridges `ToolExecutor` interface to the manager.
- **Lifecycle**: Initialized during `UnifiedToolHandler._initialize()`, cleaned up in `UnifiedToolHandler.cleanup()`.
