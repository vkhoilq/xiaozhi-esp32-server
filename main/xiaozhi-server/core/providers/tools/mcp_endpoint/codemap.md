# MCP Endpoint Tools (`core/providers/tools/mcp_endpoint/`)

## Responsibility
Implements a **remote MCP endpoint client** that connects to external MCP-compatible services over WebSocket. This allows the server to discover and invoke tools hosted on remote MCP servers, effectively extending the tool ecosystem beyond the local device and server plugins. The endpoint is configured via a URL in `config.yaml`.

## Design
- **JSON-RPC over External WebSocket**: Unlike the device MCP (which uses the device's existing WebSocket), the MCP endpoint opens a **separate WebSocket connection** to the remote endpoint URL. Uses the `websockets` library.
- **Message Listener Task**: An async task (`_message_listener`) continuously reads from the WebSocket, dispatching messages to `handle_mcp_endpoint_message()`. This provides full-duplex communication.
- **Same Protocol, Different Transport**: The protocol logic (initialize, list tools, call tool, error handling) mirrors the device MCP client closely, but the transport layer is an independent WebSocket connection managed via `MCPEndpointClient.set_websocket()` / `close()`.
- **Graceful Connection Handling**: `connect_mcp_endpoint()` validates the URL (rejects placeholder values), catches connection exceptions, and returns `None` on failure — allowing the system to continue without the endpoint.
- **Automatic Tool Registration**: When tools are received, `handle_mcp_endpoint_message()` automatically refreshes the `ToolManager` cache via `conn.func_handler.tool_manager.refresh_tools()`.

## Flow
1. **Connection** (`connect_mcp_endpoint`):
   - Opens WebSocket to the configured URL
   - Creates `MCPEndpointClient` and attaches the WebSocket
   - Starts async message listener
   - Sends initialize message (ID=1)
   - Sends `notifications/initialized`
   - Sends `tools/list` request (ID=2)
2. **Message Handling** (`handle_mcp_endpoint_message`):
   - **Result with ID in call_results**: Resolves the corresponding Future (tool call response)
   - **ID=1 (initialize)**: Logs server info from `result.serverInfo`
   - **ID=2 (tools/list)**: Processes tool array (same sanitization, description rewriting, pagination as device MCP), sets `ready=True`, refreshes tool cache
   - **Method**: Logs incoming requests from endpoint
   - **Error**: Rejects pending call Future with error message
3. **Tool Execution** (`call_mcp_endpoint_tool`):
   - Identical logic to device MCP `call_mcp_tool`: validates readiness, creates Future, parses arguments (supports JSON merge), sends `tools/call`, awaits result with 30s timeout
4. **Cleanup**: `close()` method on `MCPEndpointClient` closes the WebSocket; `UnifiedToolHandler.cleanup()` calls this during shutdown.

## Integration
- **Dependencies**: `websockets` (PyPI), `asyncio.Future`, `json`, `re`, `config.logger`
- **Configuration** (in `config.yaml`):
  ```yaml
  mcp_endpoint: "wss://your-mcp-endpoint-url/path"
  ```
- **Consumers**: `UnifiedToolHandler._initialize_mcp_endpoint()` calls `connect_mcp_endpoint()` during async initialization. `MCPEndpointExecutor` bridges the tool calls to the `ToolExecutor` interface.
- **Lifecycle**: Created during `UnifiedToolHandler._initialize()`, cleaned up in `UnifiedToolHandler.cleanup()`.
