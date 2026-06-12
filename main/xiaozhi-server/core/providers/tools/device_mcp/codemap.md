# Device MCP Tools (`core/providers/tools/device_mcp/`)

## Responsibility
Implements the **device-side MCP (Model Context Protocol) client subsystem** — enabling the server to discover and invoke tools exposed by the ESP32 client device via the JSON-RPC-based MCP protocol over WebSocket. The device acts as an MCP server, advertising its capabilities (tools, vision sampling) and responding to tool calls.

## Design
- **JSON-RPC over WebSocket**: Communication follows the MCP JSON-RPC 2.0 specification. Messages are sent as `{"type": "mcp", "payload": {...}}` over the existing device WebSocket.
- **Client State Machine**: `MCPClient` manages connection lifecycle: `uninitialized → initialized (ID=1) → tools listed (ID=2) → ready`. Uses `asyncio.Lock` for thread-safe state transitions.
- **Future-Based Async RPC**: Each tool call creates an `asyncio.Future`, registered by ID in `call_results`. The response handler resolves the future when a matching `id` is received. This provides clean async await semantics over the message-based protocol.
- **Tool Name Sanitization**: `sanitize_tool_name()` converts device tool names to safe identifiers. A `name_mapping` dictionary maintains the bidirectional mapping.
- **Caching**: `get_available_tools()` caches the formatted tool list, invalidated on each `add_tool()` call.

## Flow
1. **MCP Handshake** (initiated by server after WebSocket connects):
   - `send_mcp_initialize_message(conn)`: Sends JSON-RPC initialize with protocol version `2024-11-05`, capabilities (roots, sampling, vision with auth token)
   - Device responds with `result.serverInfo` → triggers `send_mcp_tools_list_request(conn)`
2. **Tool Discovery**:
   - `send_mcp_tools_list_request(conn)`: Sends `tools/list` request (ID=2)
   - Device responds with `result.tools` array → `handle_mcp_message()` processes each tool:
     - Extracts `name`, `description`, `inputSchema`
     - Calls `mcp_client.add_tool(new_tool)` which sanitizes and caches
     - Replaces tool names in descriptions with sanitized versions
     - Supports pagination via `nextCursor`
   - Sets `mcp_client.ready = True` → refreshes `ToolManager` cache
3. **Tool Execution** (`call_mcp_tool`):
   - Validates readiness and tool existence
   - Creates a new Future, registers by ID
   - Sends `tools/call` JSON-RPC request with parsed arguments (handles multiple JSON objects merge)
   - Awaits response with 30s timeout
   - Handles `isError`, extracts `content[0].text` if present
   - On timeout/error: cleans up the future
4. **`DeviceMCPExecutor`**: Bridges `ToolExecutor` interface to `call_mcp_tool`:
   - `execute()`: Converts arguments to JSON string, calls `call_mcp_tool`, parses response for inline `Action` directives (vision model bypass)
   - `get_tools()`: Maps `MCPClient.get_available_tools()` to `ToolDefinition` objects

## Integration
- **Dependencies**: `core.utils.util.sanitize_tool_name`, `core.utils.auth.AuthToken`, `asyncio.Future`, `json`, `re`
- **Message Format**: `{"type": "mcp", "payload": {"jsonrpc": "2.0", "id": N, "method": "...", "params": {...}}}`
- **Consumers**: `UnifiedToolHandler` creates `DeviceMCPExecutor`; the WebSocket message handler dispatches MCP payloads to `handle_mcp_message()`.
- **Vision Integration**: The initialize message includes a `vision` capability with URL and auth token, enabling on-device camera sampling.
