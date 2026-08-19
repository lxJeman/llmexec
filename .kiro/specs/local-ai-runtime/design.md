# Design Document: Local AI Runtime

## Overview

The Local AI Runtime consists of two pieces:

1. **Packager** — a developer-facing CLI that takes a GGUF model file and produces a self-contained Windows `.exe`
2. **Runtime** — the executable produced by the Packager that end users download and run

The Runtime operates in one of two modes:

- **Server mode** (default): runs headless, exposes an OpenAI-compatible HTTP API on the local network. Any client that speaks the OpenAI REST API (Continue.dev, Open WebUI, LM Studio, custom scripts) connects by simply changing its base URL.
- **Chat mode**: starts the API server identically, then additionally opens the user's default browser to a built-in web UI served from the same process.

The key technical challenge is binary portability: the `.exe` must carry the CUDA runtime, cuBLAS, and llama.cpp internally so the user never needs to install anything beyond their existing NVIDIA driver.

The inference engine is llama.cpp. It handles all token generation, quantized model loading, CUDA/CPU offloading, and context management. The Runtime sits above llama.cpp: detects hardware, configures it, exposes an API, and optionally presents a browser UI.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                      Developer machine                       │
│                                                              │
│  packager CLI                                                │
│  ├── --model model.gguf                                      │
│  ├── --name "My Model"                                       │
│  ├── --mode server|chat  (default: server)                   │
│  ├── --ui-assets ./web/  (optional, required for chat mode)  │
│  └── output: MyModel.exe                                     │
└─────────────────────────┬────────────────────────────────────┘
                          │ distributes
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                       User machine                           │
│                                                              │
│  MyModel.exe  [--chat | --server] [--port N] [--host H]      │
│  │                                                           │
│  ├── Startup: extract bundled libs to temp dir               │
│  ├── Hardware Detector                                       │
│  │     detect NVIDIA GPU + VRAM, system RAM                  │
│  ├── Configurator                                            │
│  │     select backend, calculate gpu_layers, context_size    │
│  ├── Inference Engine (llama.cpp)                            │
│  │     load model, run inference, stream tokens              │
│  ├── API Server  ──────────────────────────────────────────► HTTP
│  │     POST /v1/chat/completions  (OpenAI-compatible + SSE)  │
│  │     GET  /v1/models                                       │
│  │     GET  /health                                          │
│  │     GET  /metrics                                         │
│  │                                                           │
│  └── [chat mode only]                                        │
│        serve Web UI at GET /                                 │
│        open default browser to http://localhost:<port>       │
│                                                              │
│  External dependency: NVIDIA GPU driver (user's system)      │
└──────────────────────────────────────────────────────────────┘
```

---

## Components and Interfaces

### Packager CLI

The developer runs the Packager to produce the distributable `.exe`.

**Inputs:**
- `--model <path>` — path to a `.gguf` file (required)
- `--name <string>` — display name embedded in the output executable (required)
- `--mode server|chat` — default operating mode (default: `server`)
- `--context <int>` — optional default context size override (default: auto)
- `--ui-assets <dir>` — directory of web UI static files to bundle (required if `--mode chat`)
- `--api-key <string>` — optional API key required by clients on `/v1/` endpoints
- `--output <path>` — output executable path (default: `<name>.exe`)

**Behavior:**
1. Validates the GGUF file header (magic bytes `GGUF`, version check)
2. If `--mode chat` and `--ui-assets` is not provided: error and exit
3. Copies the GGUF file into the output bundle
4. Copies the pre-built Runtime host binary + bundled DLLs
5. Embeds web UI assets into the bundle (if provided)
6. Writes `manifest.json` (model filename, display name, default mode, default context size, optional API key hash)
7. Produces a single `.exe` (self-extracting launcher wrapping the above)

**Output structure (logical):**
```
MyModel.exe               ← launcher / self-extractor
MyModel_data/
  model.gguf              ← the GGUF model
  runtime_host.dll        ← llama.cpp host
  cublas64_12.dll
  cudart64_12.dll
  web/                    ← bundled web UI (optional)
    index.html
    ...
  manifest.json
```

---

### Hardware Detector

Runs at startup before model loading.

**Detection steps:**
1. Attempt to load `nvml.dll` dynamically
2. If NVML loads: query device 0 for GPU name and total VRAM
3. Query system RAM via `GlobalMemoryStatusEx`
4. Write results to the log

**Output:**
```
HardwareInfo {
  has_nvidia_gpu: bool
  gpu_name: String           // "NVIDIA GeForce RTX 4060"
  gpu_vram_bytes: u64
  system_ram_bytes: u64
}
```

Failure handling: any NVML failure → `has_nvidia_gpu = false`, continue with CPU mode.

---

### Configurator

Takes `HardwareInfo` + model metadata, produces `InferenceConfig`.

**GPU layer calculation:**
- Reserve 512 MB safety buffer for CUDA overhead
- `gpu_layers = floor((vram_bytes - buffer) / bytes_per_layer)`, clamped to `[0, total_layers]`

**Context size selection:**
- Standard values: `[512, 1024, 2048, 4096, 8192, 16384, 32768]`
- Select largest that fits remaining memory (VRAM if GPU, RAM if CPU)
- Cap at `manifest.default_context_size` if set

**Output:**
```
InferenceConfig {
  backend: CUDA | CPU
  gpu_layers: u32
  context_size: u32
  model_path: String
}
```

---

### Inference Engine (llama.cpp wrapper)

Wraps the llama.cpp C API.

**Key operations:**
- `load_model(config: &InferenceConfig) -> Result<Model>`
- `create_context(model: &Model, context_size: u32) -> Result<Context>`
- `generate(context: &mut Context, prompt: &str, callback: fn(token: &str)) -> Result<GenerationStats>`
  - Calls `callback` for each token as it arrives (streaming)
  - Returns `GenerationStats { tokens_generated, elapsed_ms, tokens_per_sec }`
- `stop()` — atomic flag checked after each decoded token

**llama.cpp C API surface:**
- `llama_model_load_from_file`, `llama_new_context_with_params`
- `llama_batch_init` / `llama_decode`
- `llama_sampler_chain_init` + sampling primitives
- `llama_model_free` / `llama_free`

---

### API Server

An async HTTP server (using `axum` + `tokio` in Rust) that exposes the OpenAI-compatible API.

**Endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/chat/completions` | Chat completion; supports `"stream": true` via SSE |
| GET | `/v1/models` | Lists the bundled model |
| GET | `/health` | Returns `{"status":"ok","model_loaded":true}` |
| GET | `/metrics` | Returns `{"tokens_per_sec": N, "active_requests": N, "total_requests": N}` |
| GET | `/*` | Serves bundled web UI static files (chat mode only) |

**Request queuing:** inference is single-threaded; concurrent POST requests are queued. Waiting clients receive HTTP 200 with their response once their turn arrives (not a 429).

**Authentication:** if `manifest.api_key_hash` is set, all `/v1/` requests must include `Authorization: Bearer <key>`. The `/health` and `/metrics` endpoints are unauthenticated.

**Streaming format:** Server-Sent Events with `data: <json>\n\n` lines in OpenAI delta format, terminated by `data: [DONE]\n\n`.

---

### Web UI (Chat Mode)

A static single-page application bundled by the developer at package time. The Runtime serves it from `/`.

**UI features:**
- Chat input field and send button
- Conversation history panel (rendered markdown)
- Configuration summary bar: model name, backend, GPU layers, context size
- Tokens/sec display during active generation
- Stop generation button
- Session-scoped history (in-memory, cleared on refresh)

The Web UI communicates exclusively through the local `/v1/chat/completions` endpoint using the Fetch API with streaming.

---

### Mode Selection and Startup Sequence

```
launch MyModel.exe [flags]
  │
  ├── parse flags: --chat | --server | --port | --host | --gpu-layers | --context | --backend
  ├── extract bundled libs to temp dir
  ├── Hardware Detector → HardwareInfo
  ├── Configurator (apply any flag overrides) → InferenceConfig
  ├── InferenceEngine.load_model(config)
  ├── API Server.start(port, host)
  │
  ├── if mode == chat:
  │     if web UI assets bundled:
  │       open default browser to http://localhost:<port>
  │     else:
  │       log warning "no UI assets bundled, running in server mode"
  │
  └── block on server (Ctrl+C → graceful shutdown)
```

---

## Data Models

### GGUF Validation

The Packager validates GGUF files by checking:
- Magic bytes: `0x47 0x47 0x55 0x46` ("GGUF")
- Version field: must be 2 or 3
- Tensor count and metadata KV count: must be non-zero

### Manifest JSON

```json
{
  "name": "Qwen 14B Q4",
  "model_file": "model.gguf",
  "default_mode": "server",
  "default_context_size": null,
  "api_key_hash": null,
  "version": 1
}
```

### OpenAI Chat Completion Request (subset)

```json
{
  "model": "local",
  "messages": [{"role": "user", "content": "Hello"}],
  "stream": true,
  "max_tokens": 512,
  "temperature": 0.7
}
```

### OpenAI Chat Completion Response (non-streaming)

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1234567890,
  "model": "local",
  "choices": [{
    "index": 0,
    "message": {"role": "assistant", "content": "..."},
    "finish_reason": "stop"
  }],
  "usage": {"prompt_tokens": 10, "completion_tokens": 50, "total_tokens": 60}
}
```

### SSE Streaming Delta

```
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"delta":{"content":"Hello"},"index":0}]}

data: [DONE]
```

### Log File

```
[2025-08-19 11:00:00] Hardware: RTX 4060, 8192 MB VRAM, 32768 MB RAM
[2025-08-19 11:00:00] Backend: CUDA selected
[2025-08-19 11:00:00] Config: gpu_layers=32, context=4096
[2025-08-19 11:00:01] Model loaded in 1.2s
[2025-08-19 11:00:01] API server listening on http://localhost:8080
[2025-08-19 11:00:01] Ready
```

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: GPU layer count stays within VRAM budget

*For any* GPU VRAM size and model with a known per-layer memory footprint, the Configurator's calculated `gpu_layers` value multiplied by bytes-per-layer plus the 512 MB safety buffer must never exceed the total available VRAM.

**Validates: Requirements 4.1**

---

### Property 2: GPU layer count never exceeds model layer count

*For any* model, the Configurator SHALL never assign more GPU layers than the model actually has.

**Validates: Requirements 4.1**

---

### Property 3: Selected context size fits in remaining memory

*For any* hardware configuration and gpu_layers assignment, the selected context size must fit within the remaining available memory after GPU layer allocation.

**Validates: Requirements 4.2**

---

### Property 4: Context size is a valid standard value

*For any* hardware configuration, the selected context size must be one of: 512, 1024, 2048, 4096, 8192, 16384, 32768.

**Validates: Requirements 4.2**

---

### Property 5: CPU fallback on no GPU

*For any* hardware configuration where `has_nvidia_gpu` is false, the Configurator must select the CPU backend and set `gpu_layers = 0`.

**Validates: Requirements 3.2, 4.1**

---

### Property 6: GGUF validation rejects invalid files

*For any* byte sequence that does not begin with the GGUF magic bytes and a valid version, the Packager's validator must return an error.

**Validates: Requirements 5.4, 5.5**

---

### Property 7: GGUF validation accepts valid files

*For any* well-formed GGUF file (correct magic, version 2 or 3, non-zero tensors), the Packager's validator must return success.

**Validates: Requirements 5.4**

---

### Property 8: Manifest round trip

*For any* valid manifest object, serializing it to JSON and deserializing it back must produce an equivalent object.

**Validates: Requirements 5.2, 5.3**

---

### Property 9: CUDA fallback on initialization failure

*For any* Runtime session where CUDA backend initialization raises an error, the backend must be set to CPU and `gpu_layers` must be 0 in the resulting `InferenceConfig`.

**Validates: Requirements 3.4**

---

### Property 10: API response contains required OpenAI fields

*For any* valid `/v1/chat/completions` request, the non-streaming response must contain `id`, `object`, `created`, `model`, `choices` (with at least one entry having `message.role` and `message.content`), and `usage`.

**Validates: Requirements 6.2**

---

### Property 11: Streaming response terminates with [DONE]

*For any* streaming `/v1/chat/completions` request, the SSE stream must end with a `data: [DONE]` event.

**Validates: Requirements 6.3**

---

### Property 12: /health returns 200 when model is loaded

*For any* Runtime instance where the model has been successfully loaded, `GET /health` must return HTTP 200.

**Validates: Requirements 6.5**

---

## Error Handling

| Condition | Detection | Response |
|---|---|---|
| NVML not found | `LoadLibrary("nvml.dll")` fails | Log warning, set `has_nvidia_gpu = false`, continue |
| CUDA init failure | llama.cpp context creation fails | Log error, retry with CPU, notify user |
| Model load OOM | llama.cpp returns allocation error | Display plain-language message, suggest fewer GPU layers or CPU |
| Invalid GGUF (Packager) | Magic byte check fails | Print error, exit non-zero |
| Invalid GGUF (Runtime) | Magic byte check at startup fails | Display "model file appears corrupted", exit |
| Manifest missing/malformed | JSON parse error | Use safe defaults, log warning |
| Port already in use | TCP bind fails | Report port conflict in plain language, exit |
| No UI assets in chat mode | Asset bundle absent | Log warning, continue as server-only |
| Unrecoverable panic | Top-level catch | Display log file path, exit |

---

## Testing Strategy

### Unit Tests

- GGUF validator: valid file, truncated file, wrong magic, wrong version, version 2, version 3
- Configurator: layer calculation edge cases (0 VRAM, exactly fitting, overflow), context size selection, CPU fallback
- Manifest: serialize/deserialize round trip, missing fields get defaults, extra fields are ignored
- Hardware detector: NVML unavailable path, NVML available but query failure
- API response builder: required fields present, streaming delta format, [DONE] termination
- Mode selection: flag parsing, default to server, chat mode browser-open, fallback when no UI assets

### Property-Based Tests

Using `proptest` (Rust). Each property test runs a minimum of 100 iterations.

- **Properties 1 & 2**: Random VRAM sizes (0–80 GB) and model layer configs; verify budget and cap
- **Properties 3 & 4**: After layer assignment, verify context size is valid and fits
- **Property 5**: Random configs with `has_nvidia_gpu = false`; verify CPU backend and `gpu_layers == 0`
- **Properties 6 & 7**: Random byte sequences with/without valid GGUF headers; verify validator
- **Property 8**: Random manifest objects; verify JSON round trip
- **Property 9**: Simulated CUDA init failure; verify CPU fallback
- **Properties 10 & 11**: Random valid chat request inputs; verify response shape and SSE termination
- **Property 12**: Post-load state; verify `/health` returns 200

Tag format: `// Feature: local-ai-runtime, Property N: <property title>`
