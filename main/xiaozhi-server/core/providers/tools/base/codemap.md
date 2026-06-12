# Tools Base Directory (`core/providers/tools/base/`)

## Responsibility
Defines the **core type system and abstract contracts** for the entire tool subsystem. This directory contains:
- `ToolType`: Enumeration of all tool origin categories
- `ToolDefinition`: Dataclass for structured tool metadata
- `ToolExecutor`: Abstract base class for all tool executors

It is the foundational layer that every tool executor module depends on.

## Design
- **Enum-Based Type Safety**: `ToolType` enum (`SERVER_PLUGIN`, `SERVER_MCP`, `DEVICE_IOT`, `DEVICE_MCP`, `MCP_ENDPOINT`) ensures type-safe tool categorization throughout the system.
- **Dataclass for Metadata**: `ToolDefinition` uses Python dataclasses to bundle tool name, OpenAI-format description dict, and tool type together.
- **Abstract Base Class**: `ToolExecutor` (ABC) mandates three methods:
  - `execute(conn, tool_name, arguments) → ActionResponse` — the core execution contract
  - `get_tools() → Dict[str, ToolDefinition]` — tool discovery
  - `has_tool(tool_name) → bool` — existence check
- **Action Enum Integration**: Both `ToolType` and `ToolDefinition` reference `Action` from `plugins_func.register`, creating a dependency on the action/response model.

## Key Definitions
```python
class ToolType(Enum):
    SERVER_PLUGIN = "server_plugin"    # Server-side plugin functions
    SERVER_MCP = "server_mcp"          # Server-side MCP tools
    DEVICE_IOT = "device_iot"          # Device-side IoT controls
    DEVICE_MCP = "device_mcp"          # Device-side MCP tools
    MCP_ENDPOINT = "mcp_endpoint"      # Remote MCP endpoint tools

@dataclass
class ToolDefinition:
    name: str
    description: Dict[str, Any]  # OpenAI function-calling format
    tool_type: ToolType
    parameters: Optional[Dict[str, Any]]

class ToolExecutor(ABC):
    async def execute(conn, tool_name, arguments) -> ActionResponse: ...
    def get_tools() -> Dict[str, ToolDefinition]: ...
    def has_tool(tool_name: str) -> bool: ...
```

## Integration
- **Dependencies**:
  - `plugins_func.register.Action` — for lifecycle action codes in execution results
- **Consumers**: Every tool executor (`DeviceIoTExecutor`, `ServerPluginExecutor`, `ServerMCPExecutor`, `DeviceMCPExecutor`, `MCPEndpointExecutor`), `ToolManager`, and `UnifiedToolHandler` imports from this module.
- **Exported via `__init__.py`**: `ToolType`, `ToolDefinition`, `ToolExecutor` are re-exported for convenient import.
