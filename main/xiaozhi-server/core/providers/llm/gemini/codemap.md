# core/providers/llm/gemini/

## Responsibility

Provides an LLM provider implementation for **Google Gemini** — Google's multimodal generative AI model family. This module bridges the voice assistant server with the `google-generativeai` Python SDK, supporting streaming text generation, native function/tool calling via `FunctionDeclaration`, and configurable HTTP proxy support for regions where Google APIs are not directly accessible.

## Design

### Class: `LLMProvider` (inherits `LLMProviderBase`)

**Initialization:**
- Reads `model_name` (default `"gemini-2.0-flash"`), `api_key`, optional `http_proxy` / `https_proxy`, and `timeout` from config.
- Validates the API key via `check_model_key()`.
- Tests and sets up proxy environment variables (`HTTP_PROXY`, `HTTPS_PROXY`) before configuring the Gemini SDK.
- Configures `genai.GenerativeModel` with a `GenerationConfig` (temperature=0.7, top_p=0.9, top_k=40, max_output_tokens=2048).

**Key Design Decisions:**
- **Proxy Fallback Logic** (`setup_proxy_env`): Tests HTTP and HTTPS proxies independently against `http://www.google.com` and `https://www.google.com`. If HTTPS proxy fails but HTTP proxy can reach HTTPS URLs, the HTTP proxy is reused for HTTPS. Raises `RuntimeError` if neither proxy works.
- **Native Function Calling**: `_build_tools()` converts the system's function definitions (OpenAI tool format) into Gemini `types.Tool` with `FunctionDeclaration` objects. `response_with_functions()` passes these tools to `model.generate_content()`.
- **Dialogue-to-Contents Conversion** (`_generate`): Maps `role: "assistant"` to `role: "model"` for Gemini's expected format. Handles `tool_calls` in assistant messages (converts to Gemini `function_call` parts) and `role: "tool"` messages (converts to text parts).
- **Function Call Detection**: In the streaming response, checks each `part` for `function_call` attribute. If present, yields `(None, [SimpleNamespace(...)])` with a UUID-generated ID, function name, and parsed JSON arguments. Text parts yield `(text, None)`.
- **Stream Cleanup**: `_safe_finish_stream()` provides a version-safe method to close/resolve the stream, accommodating different Gemini SDK versions.

## Flow

### Plain text flow (`response`)
1. Calls `_generate(dialogue, tools=None)`.
2. Converts dialogue to Gemini `contents` list.
3. Calls `model.generate_content(contents, stream=True)`.
4. Iterates over chunks, yielding text from each `part.text` attribute.

### Function-calling flow (`response_with_functions`)
1. Builds `tools` via `_build_tools(functions)`.
2. Calls `_generate(dialogue, tools)`.
3. In the stream, for each chunk:
   - If `part.function_call` exists: yields `(None, [tool_call])` and returns (function calls terminate further text streaming).
   - If `part.text` exists: yields `(part.text, None)`.
4. After stream completes, yields `(None, None)` as a sentinel to signal end of function mode.

## Integration

### Dependencies
- `google-generativeai` (official Google Gemini SDK)
- `requests` (proxy testing)
- `core.providers.llm.base.LLMProviderBase` (parent class)
- `config.logger` (structured logging)

### Consumed by
- The LLM provider factory/dispatch, invoked when config specifies `"type": "gemini"` (or similar alias).
- Orchestration layer calls `response()` for dialogue completion or `response_with_functions()` for intent/tool resolution.

### Notable Details
- Proxy testing is done on every initialization — this adds latency on startup but ensures connectivity.
- Function calls end the stream immediately (return after first function call chunk), as Gemini typically sends the full function declaration in one chunk.
- The `tools` parameter is passed at the top level of `generate_content`, not per-message, enabling Gemini's native function-calling mode.
