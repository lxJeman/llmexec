# Design Document: Local AI Runtime

## Overview

The Local AI Runtime consists of two pieces:

1. **Packager** — a developer-facing CLI tool that takes a GGUF model file and produces a self-contained Windows `.exe`
2. **Runtime** — the executable produced by the Packager that end users download and run

The key technical challenge is binary portability: the `.exe` must carry the CUDA runtime, cuBLAS, and llama.cpp internally so the user never needs to install anything beyond their existing NVIDIA driver.

The inference engine is llama.cpp. It handles all token generation, quantized model loading, CUDA/CPU offloading, and context management. The Runtime's job is to sit above llama.cpp: detect hardware, configure it correctly, and present the user with a usable interface.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Developer machine                   │
│                                                         │
│  packager CLI                                           │
│  ├── input: model.gguf + config                         │
│  ├── embeds: llama.cpp + CUDA runtime + cuBLAS          │
│  └── output: MyModel.exe                                │
└──────────────────────┬──────────────────────────────────┘
                       │ distributes
                       ▼
┌─────────────────────────────────────────────────────────┐
│                      User machine                       │
│                                                         │
│  MyModel.exe                                            │
│  ├── Startup                                            │
│  │   ├── extract bundled libraries to temp dir          │
│  │   └── load DLLs from temp dir                        │
│  │                                                      │
│  ├── Hardware Detector                                  │
│  │   ├── detect NVIDIA GPU + VRAM                       │
│  │   └── detect system RAM                              │
│  │                                                      │
│  ├── Configurator                                       │
│  │   ├── select backend (CUDA / CPU)                    │
│  │   ├── calculate gpu_layers                           │
│  │   └── select context size                            │
│  │                                                      │
│  ├── Inference Engine (llama.cpp)                       │
│  │   ├── load model                                     │
│  │   ├── run inference                                  │
│  │   └── stream tokens                                  │
│  │                                                      │
│  └── Chat Interface                                     │
│      ├── display config summary                         │
│      ├── accept user input                              │
│      └── stream model output                            │
│                                                         │
│  External dependency: NVIDIA GPU driver (user's system) │
└─────────────────────────────────────────────────────────┘
```

---

## Components and Interfaces

### Packager CLI

The Packager is a command-line tool the developer runs to produce the distributable `.exe`.

**Inputs:**
- `--model <path>` — path to a `.gguf` file
- `--name <string>` — display name embedded in the output executable
- `--context <int>` — optional default context size override (default: auto)
- `--output <path>` — output executable path (default: `<name>.exe`)

**Behavior:**
1. Validates the GGUF file header (magic bytes `GGUF`, version check)
2. Copies the GGUF file into the output bundle (as a sidecar or embedded resource)
3. Copies the pre-built Runtime host binary + bundled DLLs
4. Writes a small config manifest (model filename, display name, default context size) into the bundle
5. Produces a single directory or self-extracting archive that presents as one `.exe` to the user

**Output structure (logical):**
```
MyModel.exe          ← launcher / self-extractor
MyModel_data/
  model.gguf         ← the GGUF model
  runtime.dll        ← llama.cpp host
  cublas64_12.dll    ← CUDA cuBLAS
  cudart64_12.dll    ← CUDA runtime
  manifest.json      ← name, model filename, default settings
```

For true single-file distribution, the `_data/` contents are embedded into the `.exe` as a resource section and extracted to a temp directory at startup.

---

### Hardware Detector

Runs at startup before model loading.

**Detection steps:**
1. Attempt to load `nvml.dll` (NVIDIA Management Library) dynamically
2. If NVML loads: query device count, pick device 0, read GPU name and total VRAM
3. Query system RAM via `GlobalMemoryStatusEx` (Windows API)
4. Write results to the log file and display a summary line

**Output data structure:**
```
HardwareInfo {
  has_nvidia_gpu: bool
  gpu_name: string          // e.g. "NVIDIA GeForce RTX 4060"
  gpu_vram_bytes: u64       // total VRAM reported by NVML
  system_ram_bytes: u64
}
```

**Failure handling:**
- If NVML fails to load: `has_nvidia_gpu = false`, continue with CPU mode
- If NVML loads but device query fails: same as above

---

### Configurator

Takes `HardwareInfo` and model metadata, produces `InferenceConfig`.

**GPU layer calculation:**
- Query the GGUF file to determine number of layers and per-layer memory footprint
- Reserve a safety buffer (e.g. 512 MB) for CUDA overhead
- `gpu_layers = floor((vram_bytes - buffer) / bytes_per_layer)`
- Cap at total model layer count
- If `gpu_layers == total_layers`: fully GPU-accelerated
- If `gpu_layers == 0` or no GPU: CPU only

**Context size selection:**
- After GPU layers are allocated, estimate remaining available memory (VRAM if GPU, RAM if CPU)
- Choose context size from a set of standard values: 512, 1024, 2048, 4096, 8192, 16384, 32768
- Select the largest that fits within the remaining memory budget
- If the manifest specifies a default context size, use the minimum of the manifest value and the calculated maximum

**Output:**
```
InferenceConfig {
  backend: CUDA | CPU
  gpu_layers: int
  context_size: int
  model_path: string
}
```

---

### Inference Engine (llama.cpp wrapper)

Wraps the llama.cpp C API.

**Key operations:**
- `load_model(config: InferenceConfig) -> Result<Model>`
- `create_context(model: Model, context_size: int) -> Result<Context>`
- `generate(context: Context, prompt: string, callback: fn(token: string)) -> Result<()>`
  - Calls the callback for each token as it is generated (streaming)
  - Returns when generation completes or stop is requested
- `stop()` — signals the inference loop to halt after the current token

**llama.cpp C API surface used:**
- `llama_model_load_from_file`
- `llama_new_context_with_params`
- `llama_batch_init` / `llama_decode`
- `llama_sampler_chain_init` / sampling primitives
- `llama_model_free` / `llama_free`

---

### Chat Interface

A terminal-based UI (initial MVP). Runs in a loop:

1. Print hardware summary and config on startup
2. Print `> ` prompt
3. Read user input line
4. Build prompt string (including conversation history)
5. Call `generate()` with a token callback that writes each token to stdout immediately
6. After generation, print newline and tokens/sec measurement
7. Return to step 2

**Conversation history:** stored as a list of `{role, content}` pairs. Formatted using the model's chat template (read from GGUF metadata if present, otherwise use a sensible default like ChatML).

**Stop generation:** handled via a signal (Ctrl+C on Windows) that sets a flag checked by the inference loop.

**Tokens/sec measurement:** record start time when first token arrives, record end time after last token, divide total token count by elapsed seconds.

---

## Data Models

### GGUF Validation

The Packager validates GGUF files by checking:
- Magic bytes: `0x47 0x47 0x55 0x46` ("GGUF")
- Version field: must be 2 or 3 (current supported versions)
- Tensor count and metadata KV count: must be non-zero

### Manifest JSON

Written by the Packager, read by the Runtime at startup:

```json
{
  "name": "Qwen 14B Q4",
  "model_file": "model.gguf",
  "default_context_size": null,
  "version": 1
}
```

### Log File

Written to `<exe_directory>/runtime.log` or `%TEMP%/localai_<name>/runtime.log`:

```
[2025-08-19 11:00:00] Hardware: RTX 4060, 8192 MB VRAM, 32768 MB RAM
[2025-08-19 11:00:00] Backend: CUDA selected
[2025-08-19 11:00:00] Config: gpu_layers=32, context=4096
[2025-08-19 11:00:01] Model loaded in 1.2s
[2025-08-19 11:00:01] Ready
```

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: GPU layer count stays within VRAM budget

*For any* GPU VRAM size and model with a known per-layer memory footprint, the Configurator's calculated `gpu_layers` value multiplied by bytes-per-layer plus the safety buffer must never exceed the total available VRAM.

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

*For any* hardware configuration, the selected context size must be one of the standard values: 512, 1024, 2048, 4096, 8192, 16384, 32768.

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

**Validates: Requirements 5.3, 6.2**

---

### Property 9: CUDA fallback on initialization failure

*For any* Runtime session where CUDA backend initialization raises an error, the backend must be set to CPU and `gpu_layers` must be 0 in the resulting `InferenceConfig`.

**Validates: Requirements 3.4**

---

## Error Handling

| Condition | Detection | Response |
|---|---|---|
| NVML not found / no NVIDIA driver | `LoadLibrary("nvml.dll")` fails | Log warning, set `has_nvidia_gpu = false`, continue |
| CUDA init failure | `llama_backend_init` or context creation returns error | Log error, retry with CPU backend, notify user |
| Model load OOM | llama.cpp returns allocation error | Display plain-language message, suggest fewer GPU layers or CPU mode |
| Invalid GGUF (Packager) | Magic byte check fails | Print error, exit with non-zero code |
| Invalid GGUF (Runtime) | Magic byte check at startup fails | Display "model file appears corrupted", exit |
| Manifest missing or malformed | JSON parse error | Use safe defaults, log warning |
| Unrecoverable panic | Top-level catch | Display log file location, exit |

---

## Testing Strategy

### Unit Tests

- GGUF validator: valid file, truncated file, wrong magic, wrong version, version 2, version 3
- Configurator: layer calculation edge cases (0 VRAM, exactly fitting, overflow), context size selection, CPU fallback
- Manifest: serialize/deserialize round trip, missing fields get defaults, extra fields are ignored
- Hardware detector: mock NVML responses, NVML unavailable path

### Property-Based Tests

Using a property-based testing library appropriate to the implementation language (e.g., `proptest` for Rust, `hypothesis` for Python, `fast-check` for TypeScript):

Each property test runs a minimum of 100 iterations.

- **Property 1 & 2**: Generate random VRAM sizes (0 to 80 GB) and model layer configs; verify `gpu_layers * bytes_per_layer + buffer <= vram` and `gpu_layers <= total_layers`
- **Property 3 & 4**: Generate random hardware configs after layer assignment; verify context size is a valid standard value and fits in remaining memory
- **Property 5**: Generate hardware configs with `has_nvidia_gpu = false`; verify backend == CPU and `gpu_layers == 0`
- **Property 6 & 7**: Generate byte sequences with and without valid GGUF headers; verify validator output
- **Property 8**: Generate random manifest objects; verify JSON round trip
- **Property 9**: Simulate CUDA init failures; verify CPU fallback is applied

Tag format for each test:
`// Feature: local-ai-runtime, Property N: <property title>`
