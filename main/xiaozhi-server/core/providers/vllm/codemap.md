# VLLM (Vision-Language Large Model) Provider

## Responsibility

Provides vision-language understanding capability — processes images (received as base64-encoded JPEG from ESP32 clients) along with text questions and returns text responses. Used for visual question answering when the device has a camera.

## Design

- **Abstract Base Class**: `VLLMProviderBase` defines single method:
  - `response(question, base64_image)` — generator/iterator returning text response

- **OpenAI-Compatible Implementation** (`openai.py`):
  - Uses `openai.OpenAI` client with configurable `base_url` (supports any OpenAI-compatible API)
  - Configurable parameters: `max_tokens` (default 500), `temperature` (default 0.7), `top_p` (default 1.0)
  - Builds a multimodal message: text question + base64 JPEG image
  - Appends "(请使用中文回复)" to the question
  - Uses `stream=False` (synchronous completion)
  - Validates API key via `check_model_key()` utility

## Flow

1. `response(question, base64_image)` called with user question and camera image
2. Appends Chinese-language instruction to question
3. Builds OpenAI-compatible message array with text and image_url content
4. Calls `client.chat.completions.create()` with model name and messages
5. Returns `choices[0].message.content`

## Integration

- **Consumer**: Connection handler calls `vllm_provider.response()` when a camera image is available and VLLM mode is active
- **Dependency**: `openai` Python package
- **Dependency**: `core.utils.util.check_model_key()` — validates API key presence
- **Configuration**: Model name, API key, base URL, and generation parameters from config
