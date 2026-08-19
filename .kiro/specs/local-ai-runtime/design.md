# Design Document: Local AI Runtime

## Overview

The Local AI Runtime consists of two pieces:

1. **Packager** — a developer-facing CLI that takes a GGUF model file and produces a self-contained Windows `.exe`
2. **Runtime** — the executable produced by the Packager that end users download and run

The Runtime operates in one of two modes:

- **Server mode** (default): runs headless, exposes an OpenAI-compatible HTTP API on the local network.
- **Chat mode**: starts the API server identically, then additionally opens the user's default browser to a built-in web UI served from the same process.

**MVP scope: Windows only.** Cross-platform support (Linux, macOS, Vulkan, Metal) is deferred to Phase 2.

**CUDA DLL redistribution:** `cudart64_12.dll` and `cublas64_12.dll` are redistributable under the NVIDIA CUDA Toolkit EULA. The developer building the Packager must have the CUDA 12.x Toolkit installed; the Packager copies these DLLs from the local SDK installation at build time.

**llama.cpp bindings:** The project uses the `llama-cpp-rs` crate. This crate provides safe Rust bindings and handles its own compilation of the llama.cpp C++ source via `build.rs`.

---

## Architecture

```
+--------------------------------------------------------------+
|                      Developer machine                       |
|                                                              |
|  packager CLI                                                |
|  +-- --model model.gguf                                      |
|  +-- --name "My Model"                                       |
|  +-- --mode server|chat  (default: server)                   |
|  +-- --ui-assets ./web/  (optional, required for chat mode)  |
|  +-- output: MyModel.exe                                     |
+-------------------------+------------------------------------+
                          | distributes
                          v
+--------------------------------------------------------------+
|                    User machine (Windows)                    |
|                                                              |
|  MyModel.exe  [--chat | --server] [--port N] [--host H]      |
|  |                                                           |
|  +-- Self-Extractor: extract DLLs from .exe resources        |
|  |     -> %TEMP%/localai_<name>/<exe_hash>/                  |
|  +-- Hardware Detector                                       |
|  |     detect NVIDIA GPU + VRAM, system RAM                  |
|  +-- Configurator                                            |
|  |     select backend, calculate gpu_layers, context_size    |
|  +-- Inference Engine (llama.cpp via llama-cpp-rs)           |
|  |     load model, run inference, stream tokens              |
|  +-- API Server (axum + tokio, async request queue channel)  |
|  |     POST /v1/chat/completions  (OpenAI + SSE)             |
|  |     GET  /v1/models                                       |
|  |     GET  /health                                          |
|  |     GET  /metrics                                         |
|  |                                                           |
|  +-- [chat mode only]                                        |
|        serve Web UI at GET /                                 |
|        open browser: cmd /c start http://localhost:<port>    |
|                                                              |
|  External dependency: NVIDIA GPU driver (user's system)      |
+--------------------------------------------------------------+
```

---

## Components and Interfaces

### Self-Extractor (first step at startup)

The Self-Extractor runs before any other component. DLLs must be on disk before `LoadLibrary` can find them.

**Behavior:**
1. Compute extraction directory: `%TEMP%\localai_<name>\<sha256_of_exe_first_64kb>\`
2. If directory exists and `extracted.ok` sentinel file is present: skip (fast path for repeat launches)
3. Otherwise: read each DLL from the `.exe` Win32 resource section (`FindResource` / `LoadResource` / `LockResource`), write to disk, then write `extracted.ok`
4. Call `SetDllDirectoryW` to prepend the extraction directory to the DLL search path

---

### Packager CLI

**Inputs:**
- `--model <path>` — path to a `.gguf` file (required)
- `--name <string>` — display name (required)
- `--mode server|chat` — default mode (default: `server`)
- `--context <int>` — optional default context size override
- `--ui-assets <dir>` — web UI static files directory (required if `--mode chat`)
- `--api-key <string>` — optional API key; stored as `sha256:<hex_salt>:<hex_digest>`
- `--output <path>` — output path (default: `<name>.exe`)

**Behavior:**
1. Validate GGUF header (magic, version 2 or 3, non-zero tensor count)
2. If `--mode chat` and `--ui-assets` not provided: print error to stderr, exit non-zero
3. Embed GGUF model, DLLs (from local CUDA SDK), and web assets as Win32 resource sections
4. Write `manifest.json` and embed as a resource
5. Produce the single `.exe`

---

### Hardware Detector

Runs after self-extraction, before model loading.

**Detection steps:**
1. Attempt to load `nvml.dll` via `LoadLibraryW`
2. If NVML loads: call `nvmlInit_v2`, query device 0 for GPU name and total VRAM via `nvmlDeviceGetMemoryInfo`
3. Query system RAM via `GlobalMemoryStatusEx`
4. Write results to the log

**Output type:**
```rust
struct HardwareInfo {
    has_nvidia_gpu: bool,
    gpu_name: String,
    gpu_vram_bytes: u64,
    system_ram_bytes: u64,
}
```

Any NVML failure (load, init, or query) sets `has_nvidia_gpu = false` and continues.

---

### Configurator

Takes `HardwareInfo` + model metadata from the GGUF file, produces `InferenceConfig`.

**Deriving `bytes_per_layer`:**
Read the per-tensor metadata from the GGUF file's tensor descriptors and sum the sizes of the attention and feed-forward tensors for one layer. If the metadata is absent or malformed, fall back to a conservative 200 MB/layer estimate.

**GPU layer calculation:**
```
available = vram_bytes - 512 MB
if available <= 0: gpu_layers = 0
else: gpu_layers = min(floor(available / bytes_per_layer), total_model_layers)
```

**Context size selection:**
```
STANDARD_SIZES = [512, 1024, 2048, 4096, 8192, 16384, 32768]
remaining = vram_bytes - (gpu_layers * bytes_per_layer)   // or system_ram for CPU
bytes_per_token ≈ 512 bytes (conservative estimate)
selected = largest STANDARD_SIZE where size * bytes_per_token <= remaining
if manifest.default_context_size is set: selected = min(selected, manifest.default_context_size)
```

**Context creation OOM recovery:**
If `create_session` fails with OOM after model load succeeded, halve `context_size` and retry until either success or `context_size < 512`. If all retries fail: display plain-language OOM message and exit.

**Output type:**
```rust
struct InferenceConfig {
    backend: Backend,    // CUDA | CPU
    gpu_layers: u32,
    context_size: u32,
    model_path: String,
    bytes_per_layer: u64,
}
```

---

### Inference Engine (llama-cpp-rs)

Uses the `llama_cpp` crate's session-based API.

**Key operations:**
- `load_model(config: &InferenceConfig) -> Result<LlamaModel>`
- `create_session(model: &LlamaModel, context_size: u32) -> Result<LlamaSession>`
- `generate(session: &mut LlamaSession, prompt: &str, stop_flag: Arc<AtomicBool>, callback: impl Fn(&str)) -> Result<GenerationStats>`
  - Calls `callback` per token; checks `stop_flag` after each token
  - Returns `GenerationStats { tokens_generated: u32, elapsed_ms: u64, tokens_per_sec: f32 }`
- OOM from llama.cpp → `Error::OutOfMemory` with plain-language message

**Chat template:**
- Read `tokenizer.chat_template` key from GGUF metadata if present
- Fall back to ChatML format if absent:
  ```
  <|im_start|>system\n{system}\n<|im_end|>\n
  <|im_start|>user\n{content}\n<|im_end|>\n
  <|im_start|>assistant\n
  ```
- `format_messages(messages: &[Message], template: &ChatTemplate) -> String`

---

### API Server

Async HTTP server using `axum` + `tokio`.

**Concurrency model — request queue channel (not a mutex):**
- A `tokio::sync::mpsc` channel is the inference queue
- Each `/v1/chat/completions` handler sends `(request, oneshot::Sender<Response>)` to the channel
- A `spawn_blocking` worker receives from the channel and calls the synchronous `generate()`
- The handler awaits the `oneshot::Receiver<Response>` and streams the result back

This avoids blocking async threads on a mutex held for the full duration of inference.

**Endpoints:**

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/v1/chat/completions` | Yes (if key set) | Chat completion + SSE streaming |
| GET | `/v1/models` | Yes (if key set) | Lists bundled model |
| GET | `/health` | No | `{"status":"ok","model_loaded":true}` |
| GET | `/metrics` | No | `{"tokens_per_sec":N,"active_requests":N,"total_requests":N}` |
| GET | `/*` | No | Static web UI files; 404 if no assets bundled |

**Authentication:** Check `Authorization: Bearer <key>`. Compute `SHA-256(stored_salt || provided_key)` and compare to stored digest using `subtle::ConstantTimeEq` to prevent timing attacks.

**Streaming:** SSE with `data: <json>\n\n` per token, terminated by `data: [DONE]\n\n`.

---

### Web UI (Chat Mode)

Static single-page application (HTML + CSS + vanilla JS, no build step). Bundled by the developer; served from `/`.

**Features:**
- Chat input and send button
- Conversation history with markdown rendering (`marked.js`, bundled inline)
- Configuration summary bar (fetched from `/health`)
- Tokens/sec display during generation
- Stop button via `AbortController.abort()`
- In-memory session history

---

### Mode Selection and Startup Sequence

```
launch MyModel.exe [flags]
  |
  +-- Self-Extractor: extract DLLs to %TEMP%/localai_<name>/<hash>/
  +-- parse flags: --chat | --server | --port | --host | --gpu-layers | --context | --backend
  +-- Hardware Detector -> HardwareInfo
  +-- Configurator (apply flag overrides) -> InferenceConfig
  +-- InferenceEngine.load_model(config)       [OOM -> plain-language message, exit]
  +-- InferenceEngine.create_session(...)      [OOM -> halve context, retry; if min fails, exit]
  +-- start inference queue worker task
  +-- API Server.start(port, host)             [port conflict -> plain-language message, exit]
  |
  +-- if mode == chat:
  |     if web UI assets bundled:
  |       cmd /c start http://localhost:<port>
  |     else:
  |       log WARNING "No UI assets bundled, running in server-only mode"
  |
  +-- register Ctrl+C handler:
  |     stop accepting new requests
  |     wait up to 30s for in-flight inference to complete
  |     cancel queued requests, close connections
  |     free llama model + session
  |     exit 0
  |
  +-- block on server
```

---

## Data Models

### GGUF Validation

Check at byte offsets (all little-endian):
- Offset 0, 4 bytes: magic `0x47 0x47 0x55 0x46`
- Offset 4, 4 bytes (u32): version must be 2 or 3
- Offset 8, 8 bytes (u64): tensor count must be non-zero
- Offset 16, 8 bytes (u64): metadata KV count must be non-zero

### Manifest JSON

```json
{
  "name": "Qwen 14B Q4",
  "model_file": "model.gguf",
  "default_mode": "server",
  "default_context_size": null,
  "api_key_hash": null,
  "schema_version": 1
}
```

Deserialization defaults (forward-compatibility):
- `default_mode` missing → `"server"`
- `default_context_size` missing → `null`
- `api_key_hash` missing → `null`
- `schema_version` missing or 0 → treat as `1`, log warning
- `schema_version > 1` → log warning, attempt to run with known fields

### OpenAI Chat Completion Request

```json
{
  "model": "local",
  "messages": [{"role": "user", "content": "Hello"}],
  "stream": false,
  "max_tokens": 512,
  "temperature": 0.7
}
```

### OpenAI Chat Completion Response (non-streaming)

```json
{
  "id": "chatcmpl-<uuid>",
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
data: {"id":"chatcmpl-<uuid>","object":"chat.completion.chunk","choices":[{"delta":{"content":"Hello"},"index":0,"finish_reason":null}]}

data: {"id":"chatcmpl-<uuid>","object":"chat.completion.chunk","choices":[{"delta":{},"index":0,"finish_reason":"stop"}]}

data: [DONE]
```

### Log Lines

```
[2025-08-19 11:00:00] [INFO]  Hardware: RTX 4060, 8192 MB VRAM, 32768 MB RAM
[2025-08-19 11:00:00] [INFO]  Backend: CUDA selected (NVIDIA GPU detected)
[2025-08-19 11:00:00] [INFO]  Backend: CUDA, GPU layers: 32/32, Context: 4096 tokens, VRAM: 8192 MB
[2025-08-19 11:00:01] [INFO]  Model loaded in 1.2s
[2025-08-19 11:00:01] [INFO]  API server listening on http://localhost:8080
[2025-08-19 11:00:01] [INFO]  Ready
[2025-08-19 11:00:05] [INFO]  POST /v1/chat/completions 200 (3.2s, 42 tokens, 13.1 tok/s)
```

Log destination: `<exe_dir>/runtime.log`; fallback to `%TEMP%/localai_<name>/runtime.log` if not writable.

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: GPU layer allocation satisfies VRAM budget and model layer cap

*For any* VRAM size (including 0), per-layer byte cost, and total model layer count, the Configurator's `gpu_layers` must satisfy both constraints simultaneously:
`gpu_layers * bytes_per_layer + 512 MB <= vram_bytes` AND `gpu_layers <= total_model_layers`.

**Validates: Requirements 4.1**

---

### Property 2: Selected context size is a valid standard value that fits remaining memory

*For any* hardware configuration and GPU layer assignment, the selected context size must be a member of `{512, 1024, 2048, 4096, 8192, 16384, 32768}` and `context_size * bytes_per_token <= remaining_memory`.

**Validates: Requirements 4.2**

---

### Property 3: CPU fallback on no GPU

*For any* `HardwareInfo` where `has_nvidia_gpu = false`, the Configurator must produce `backend = CPU` and `gpu_layers = 0`.

**Validates: Requirements 3.2, 4.1**

---

### Property 4: GGUF validator correctly classifies all inputs

*For any* byte sequence that lacks the magic bytes `0x47 0x47 0x55 0x46` at offset 0, or has a version field other than 2 or 3, the validator must return an error. *For any* byte sequence with valid magic, version 2 or 3, and non-zero tensor and KV counts, the validator must return success.

**Validates: Requirements 5.4, 5.5**

---

### Property 5: Manifest round trip

*For any* valid `Manifest` object, serializing to JSON and deserializing back must produce a structurally equivalent object with all fields preserved.

**Validates: Requirements 5.6**

---

### Property 6: CUDA init failure produces valid CPU config

*For any* CUDA initialization failure, the resulting `InferenceConfig` must have `backend = CPU` and `gpu_layers = 0`.

**Validates: Requirements 3.4**

---

### Property 7: API non-streaming response contains all required OpenAI fields

*For any* valid `/v1/chat/completions` request with `stream: false`, the response must contain `id`, `object`, `created`, `model`, `choices` (at least one entry with `message.role` and `message.content`), and `usage`.

**Validates: Requirements 6.2**

---

### Property 8: Streaming response terminates with [DONE] after valid delta lines

*For any* `/v1/chat/completions` request with `stream: true`, every `data:` line before the final one must be a valid JSON object, and the final line must be `data: [DONE]`.

**Validates: Requirements 6.3**

---

### Property 9: /health returns 200 after model load

*For any* Runtime instance where the model has loaded successfully, `GET /health` returns HTTP 200 with body `{"status":"ok","model_loaded":true}`.

**Validates: Requirements 6.5**

---

### Property 10: Chat template format/parse round trip

*For any* list of chat messages, `format_messages` followed by parsing the output must recover the original role and content for each message.

**Validates: Requirements 6.2 (via chat template correctness)**

---

## Error Handling

| Condition | Detection | Response |
|---|---|---|
| NVML not found | `LoadLibraryW("nvml.dll")` fails | Log warning, `has_nvidia_gpu = false`, continue |
| CUDA init failure | llama.cpp context creation fails | Log error, `backend = CPU`, `gpu_layers = 0`, notify user |
| Model load OOM | llama.cpp allocation error | Plain-language message, suggest fewer GPU layers or CPU, exit |
| Context creation OOM | `create_session` returns OOM | Halve context size, retry; if 512 still fails, plain-language message, exit |
| Invalid GGUF (Packager) | Magic/version check fails | Descriptive error to stderr, exit non-zero, no output file |
| Invalid GGUF (Runtime) | Magic check at startup | "Model file appears corrupted", log path, exit |
| Manifest malformed | JSON parse error | Use safe defaults, log warning |
| `schema_version` unknown | Value > 1 | Log warning, attempt run with known fields |
| Port in use | TCP bind `EADDRINUSE` | Plain-language message with port number, exit non-zero |
| No UI assets in chat mode | Asset bundle absent | Log warning, continue as server-only |
| Unrecoverable panic | `catch_unwind` | Print log path and brief message to stderr, exit non-zero |
| Ctrl+C / SIGINT | OS signal | Drain in-flight request (max 30s), cancel queue, free resources, exit 0 |

---

## Testing Strategy

### Unit Tests

- GGUF validator: valid file, truncated file, wrong magic, wrong version, version 2 and 3
- Configurator: 0 VRAM, exactly-fitting VRAM, overflow; context size at each memory tier; CPU fallback; context OOM retry halving logic
- Manifest: round trip, missing fields get defaults, extra fields ignored, unknown `schema_version` logs warning
- Hardware detector: NVML unavailable, NVML loaded but query failure
- API response builder: required fields present, streaming delta format, `[DONE]` termination
- Mode selection: all flag combinations, default to server, chat fallback when no UI assets
- Chat template: ChatML fallback, `format_messages` output shape
- Graceful shutdown: `stop_flag` halts generation, queued requests are cancelled
- CLI flags: all flags parse correctly, overrides apply to `InferenceConfig`

### Property-Based Tests

Using `proptest`. Each test runs a minimum of 100 iterations.

- **Property 1**: Random VRAM (0–80 GB), `bytes_per_layer` (50–500 MB), `total_layers` (1–128); verify both constraints
- **Property 2**: Random hardware configs after layer assignment; verify context value and memory fit
- **Property 3**: Random `HardwareInfo` with `has_nvidia_gpu = false`; verify `backend = CPU`, `gpu_layers = 0`
- **Property 4**: Random byte sequences with/without valid GGUF headers; verify correct accept/reject
- **Property 5**: Random `Manifest` structs; verify JSON round trip
- **Property 6**: Simulated CUDA init failure; verify `backend = CPU`, `gpu_layers = 0`
- **Properties 7 & 8**: Random valid request inputs; verify response shape and SSE termination
- **Property 9**: Post-load state; verify `/health` response
- **Property 10**: Random message lists; verify `format_messages` → parse round trip

Tag format: `// Feature: local-ai-runtime, Property N: <property title>`
