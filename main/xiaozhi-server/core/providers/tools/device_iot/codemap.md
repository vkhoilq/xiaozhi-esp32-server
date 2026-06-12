# Device IoT Tools (`core/providers/tools/device_iot/`)

## Responsibility
Provides the **device-side IoT tool subsystem** — enabling LLM-driven control of physical IoT devices connected to the ESP32 client. The server receives device capability descriptors from the client, registers them as callable tools, and relays commands back to the device over the WebSocket. Supports both property queries (e.g., get temperature) and method invocations (e.g., turn on light).

## Design
- **Dynamic Tool Registration**: Devices self-describe their capabilities via `descriptor` messages (name, properties, methods). The `handleIotDescriptors` function creates `IotDescriptor` objects and registers corresponding tools dynamically via `DeviceIoTExecutor.register_iot_tools()`.
- **Descriptive vs. Control Separation**: Each device gets two categories of tools:
  - **Query tools** (`get_{device}_{property}`): Read property values from in-memory state
  - **Control tools** (`{device}_{method}`): Send commands to the device over WebSocket
- **Naming Convention**: Tools follow `get_{devicename}_{property}` for queries and `{devicename}_{method}` for controls, enabling `DeviceIoTExecutor.execute()` to parse the tool name to determine operation type.
- **Response Template System**: LLM function call parameters include `response_success` and `response_failure` template strings with `{value}` placeholders, allowing the LLM to customize the verbal response.
- **State Tracking**: `handleIotStatus` updates in-memory property values when the device reports state changes, keeping the query path current.

## Flow
1. **Device Descriptor Registration**:
   - Device sends `{"type": "iot", "descriptors": [...]}` message
   - `handleIotDescriptors()` waits for `conn.func_handler` initialization (up to 5s)
   - Creates `IotDescriptor` objects, stores in `conn.iot_descriptors`
   - For descriptors missing `properties`, auto-extracts from method parameters
   - Calls `conn.func_handler.register_iot_tools(descriptors)` → `DeviceIoTExecutor.register_iot_tools()`
   - Generates query tools for each property and control tools for each method
   - Refreshes `ToolManager` cache
2. **Device State Update**:
   - Device sends `{"type": "iot", "states": [...]}` message
   - `handleIotStatus()` matches device name, updates property values with type validation
3. **Tool Execution** (`DeviceIoTExecutor.execute`):
   - Parses tool name to determine query (`get_`) or control (everything else)
   - **Query**: Looks up property value from `conn.iot_descriptors`, applies `response_success` template
   - **Control**: Extracts control parameters (excluding response templates), sends `{"type":"iot","commands":[{"name":"device","method":"method","parameters":{}}]}` over WebSocket, waits 100ms, returns `response_success` template

## Integration
- **Dependencies**: `config.logger`, `plugins_func.register.Action/ActionResponse`, `..base.ToolType/ToolDefinition/ToolExecutor`
- **Messages to Device**: `{"type": "iot", "commands": [{"name": "<device>", "method": "<method>", "parameters": {...}}]}`
- **Messages from Device**: `{"type": "iot", "descriptors": [...]}`, `{"type": "iot", "states": [...]}`
- **Consumers**: `UnifiedToolHandler` creates `DeviceIoTExecutor`, registers IoT tools when descriptors arrive, and delegates tool calls during LLM function call handling.
