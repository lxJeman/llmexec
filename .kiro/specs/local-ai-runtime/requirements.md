# Requirements Document

## Introduction

A tool that packages a specific LLM (in GGUF format), llama.cpp, and the CUDA runtime into a distributable Windows application. The developer uses the **Packager** CLI to produce a self-contained `.exe`; end users download that `.exe` and run it with no prior setup.

The Runtime supports two operating modes chosen at launch via a flag:

- **Server mode** (default): runs headless and exposes an OpenAI-compatible HTTP API on the local network, allowing any OpenAI-compatible client to connect
- **Chat mode**: additionally opens a browser-based chat UI served from the same process

The developer may optionally bundle a pre-built web UI into the `.exe` at package time. When the Runtime launches in chat mode, it opens the user's default browser to the local UI automatically.

The initial target is Windows with NVIDIA GPU (CUDA), with Linux, macOS, AMD (Vulkan), and Apple Silicon (Metal) support added in later phases. The MVP targets Windows only; cross-platform paths are out of scope until Phase 2.

---

## Glossary

- **Packager**: The developer-facing CLI tool that bundles a model, llama.cpp, and the CUDA runtime into a distributable `.exe`
- **Runtime**: The self-contained executable produced by the Packager that the end user downloads and runs
- **Server mode**: Runtime operating mode where the process runs headless and serves the HTTP API only
- **Chat mode**: Runtime operating mode where the HTTP API is served and the browser-based UI is opened automatically
- **Web UI**: An optional browser-based chat interface bundled by the developer at package time
- **Backend**: The GPU/CPU compute library used for inference (CUDA for NVIDIA, CPU as fallback for MVP)
- **Model**: A GGUF-format language model file used for inference
- **Inference Engine**: The embedded llama.cpp instance used for token generation
- **GGUF**: The model file format used by llama.cpp
- **Quantization**: A compression technique that reduces model size and memory usage (e.g., Q4_K_M, Q8_0)
- **GPU Layers**: The number of model layers offloaded to GPU VRAM for acceleration
- **Bytes Per Layer**: The memory cost of a single model layer, derived from the model's tensor metadata in the GGUF file at load time
- **Context Size**: The maximum number of tokens the model can process in one session
- **Hardware Detector**: The Runtime subsystem that identifies the user's GPU, VRAM, and available system RAM at startup
- **Configurator**: The Runtime subsystem that selects GPU layer count and context size based on detected hardware
- **API Server**: The Runtime subsystem that handles HTTP requests and serves the OpenAI-compatible REST API
- **Self-Extractor**: The launcher stub embedded in the `.exe` that extracts bundled DLLs to a temp directory before any llama.cpp calls

---

## Requirements

### Requirement 1: Single-Executable Distribution

**User Story:** As a non-technical user, I want to download and run a single file without any prior setup, so that I can use a local AI model without installing Python, CUDA, or any other dependency.

#### Acceptance Criteria

1. THE Runtime SHALL be distributed as a single `.exe` file on Windows
2. THE Runtime SHALL bundle all required libraries including the CUDA runtime, cuBLAS, and llama.cpp as embedded resource sections within the executable
3. WHEN the Runtime is launched, THE Self-Extractor SHALL extract all bundled DLLs to `%TEMP%/localai_<name>/` before any llama.cpp calls are made
4. IF the host system does not have a compatible NVIDIA GPU driver installed, THEN THE Runtime SHALL fall back to CPU inference and notify the user in plain language
5. THE Runtime SHALL NOT require administrator privileges to run

---

### Requirement 2: Automatic Hardware Detection

**User Story:** As a user, I want the application to detect my GPU and configure itself automatically, so that I don't have to specify GPU settings, memory limits, or backends.

#### Acceptance Criteria

1. WHEN the Runtime starts, THE Hardware Detector SHALL identify whether an NVIDIA GPU is present and report the GPU model name and available VRAM
2. WHEN the Runtime starts, THE Hardware Detector SHALL identify total system RAM as a fallback memory reference
3. IF no NVIDIA GPU is detected, THEN THE Hardware Detector SHALL report CPU-only mode to the user
4. THE Hardware Detector SHALL complete detection before model loading begins

---

### Requirement 3: Automatic Backend Selection

**User Story:** As a user, I want the runtime to automatically choose CUDA or CPU without me needing to know what those are, so that I get the best available performance.

#### Acceptance Criteria

1. WHEN an NVIDIA GPU with a compatible driver is detected, THE Configurator SHALL select the CUDA backend
2. WHEN no compatible NVIDIA GPU is detected, THE Configurator SHALL select the CPU backend
3. THE Configurator SHALL log the selected backend and the reason for selection to the log file
4. IF the CUDA backend fails to initialize at runtime, THEN THE Configurator SHALL automatically fall back to the CPU backend, set `gpu_layers` to 0, and notify the user with a message stating which backend was attempted and why it failed

---

### Requirement 4: Automatic Model Configuration

**User Story:** As a user, I want the runtime to automatically determine how to load the model given my hardware, so that I don't have to understand GPU layers, quantization, or context sizes.

#### Acceptance Criteria

1. WHEN the Runtime starts, THE Configurator SHALL derive the per-layer memory cost from the model's GGUF tensor metadata, then calculate the maximum number of GPU layers that fit within available VRAM minus a 512 MB safety buffer, clamped to the range [0, total_model_layers]
2. WHEN the Runtime starts, THE Configurator SHALL select the largest context size from the set [512, 1024, 2048, 4096, 8192, 16384, 32768] that fits within remaining memory after GPU layer allocation
3. THE Configurator SHALL write a log line at INFO level to the structured log file before inference begins in the format: "Backend: <CUDA|CPU>, GPU layers: <n>/<total>, Context: <n> tokens, VRAM: <n> MB"
4. WHERE an advanced user wants manual control, THE Runtime SHALL accept optional command-line flags to override GPU layer count, context size, and backend

---

### Requirement 5: Model Bundling by the Developer

**User Story:** As a developer, I want to bundle a specific GGUF model into the Runtime executable, so that my end users receive a single file that contains everything needed to run that model.

#### Acceptance Criteria

1. THE Packager SHALL accept a GGUF model file as input and produce a single `.exe` that contains the model, llama.cpp, and the CUDA runtime
2. THE Packager SHALL accept the following configuration inputs: model file path (required), display name (required), default operating mode `server` or `chat` (default: `server`), optional default context size, optional web UI assets directory, optional API key, and output path
3. WHEN the Runtime is built, THE Packager SHALL embed the GGUF model data and all DLLs as resource sections in the output `.exe`
4. THE Packager SHALL validate that the provided model file begins with the GGUF magic bytes `0x47 0x47 0x55 0x46` and has a version field of 2 or 3 before producing the output executable
5. IF the provided model file fails GGUF validation, THEN THE Packager SHALL print a descriptive error message to stderr and exit with a non-zero exit code without producing any output file
6. THE Packager SHALL write a `manifest.json` into the bundle containing: `name`, `model_file`, `default_mode`, `default_context_size`, `api_key_hash` (SHA-256 with random salt, or null), `version`, and `schema_version`

---

### Requirement 6: Server Mode — OpenAI-Compatible HTTP API

**User Story:** As a developer or power user, I want the runtime to expose an OpenAI-compatible HTTP API on the local network, so that I can connect any existing OpenAI-compatible client or tool to my local model without any custom integration work.

#### Acceptance Criteria

1. WHEN the Runtime starts in server mode, THE API Server SHALL listen on a configurable port (default 8080) on localhost and optionally on all network interfaces
2. THE API Server SHALL implement `POST /v1/chat/completions` accepting an OpenAI-compatible request body and returning an OpenAI-compatible response containing `id`, `object`, `created`, `model`, `choices`, and `usage` fields
3. WHEN a client requests streaming via `"stream": true`, THE API Server SHALL return a Server-Sent Events stream of incremental tokens in OpenAI delta format, terminated by a final `data: [DONE]` event
4. THE API Server SHALL implement `GET /v1/models` returning the bundled model's name and metadata
5. THE API Server SHALL implement `GET /health` returning HTTP 200 with a JSON body `{"status":"ok","model_loaded":true}` when the model is loaded
6. THE API Server SHALL implement `GET /metrics` returning current inference statistics including tokens per second and active request count
7. WHEN multiple requests arrive concurrently, THE API Server SHALL queue them and process one at a time; each waiting client SHALL receive its HTTP 200 response when its turn arrives, with no timeout imposed by the server
8. WHERE the developer configures an API key at package time, THE API Server SHALL require that key as a Bearer token on all `/v1/` endpoints; the `/health` and `/metrics` endpoints SHALL remain unauthenticated

---

### Requirement 7: Chat Mode — Browser UI

**User Story:** As a non-technical user, I want to open a browser and interact with the model through a chat interface, so that I can use it without understanding HTTP APIs or terminals.

#### Acceptance Criteria

1. WHEN the Runtime is launched with the `--chat` flag or was packaged with `--mode chat`, THE Runtime SHALL start the API Server and then open the user's default browser to `http://localhost:<port>`
2. THE Web UI SHALL display a chat input field, a conversation history panel with markdown rendering, and a configuration summary showing model name, backend, GPU layers, and context size
3. WHEN the user submits a message, THE Web UI SHALL stream the model's response token by token as it arrives using the Fetch API with `ReadableStream`
4. THE Web UI SHALL display tokens-per-second during active generation
5. THE Web UI SHALL provide a stop button that aborts the in-progress fetch request
6. THE Web UI SHALL maintain conversation history for the current session in memory, cleared on page refresh
7. WHERE the developer did not bundle web UI assets, THEN THE Runtime SHALL log a warning and fall back to server-only mode rather than opening a browser page

---

### Requirement 8: Runtime Mode Selection

**User Story:** As an operator or developer, I want to control whether the runtime runs as a headless server or opens a browser UI, so that I can deploy it appropriately for its use case.

#### Acceptance Criteria

1. THE Runtime SHALL default to server mode when launched without any mode flag
2. WHEN launched with `--chat`, THE Runtime SHALL enter chat mode
3. WHEN launched with `--server`, THE Runtime SHALL enter server mode regardless of the packaged default
4. THE Runtime SHALL accept `--port <n>` to override the default listen port
5. WHEN launched with `--host 0.0.0.0`, THE Runtime SHALL bind to all network interfaces instead of only localhost

---

### Requirement 9: Graceful Shutdown

**User Story:** As an operator, I want the runtime to shut down cleanly when I press Ctrl+C, so that in-flight requests are not abruptly terminated and resources are released.

#### Acceptance Criteria

1. WHEN the Runtime receives a SIGINT or Ctrl+C signal, THE Runtime SHALL stop accepting new requests immediately
2. WHEN shutdown is triggered, THE Runtime SHALL wait for any currently executing inference request to complete before exiting, up to a maximum of 30 seconds
3. WHEN shutdown is triggered, THE Runtime SHALL cancel any requests waiting in the queue and close their connections
4. WHEN shutdown completes, THE Runtime SHALL release all llama.cpp model and context resources before the process exits

---

### Requirement 10: Error Handling and User Feedback

**User Story:** As a non-technical user, I want clear, plain-language feedback when something goes wrong, so that I understand what happened and what I can do.

#### Acceptance Criteria

1. IF model loading fails due to insufficient memory, THEN THE Runtime SHALL display a plain-language message explaining the issue and suggest reducing GPU layers or switching to CPU
2. IF the CUDA backend fails to initialize, THEN THE Runtime SHALL display which backend was attempted, why it failed, and confirm the fallback being used
3. THE Runtime SHALL write timestamped structured log lines to `<exe_dir>/runtime.log`; if that path is not writable, THE Runtime SHALL fall back to `%TEMP%/localai_<name>/runtime.log`
4. IF the Runtime encounters an unrecoverable error, THEN THE Runtime SHALL print the log file path and a brief plain-language message to stderr before exiting rather than silently crashing
5. IF the requested port is already in use, THEN THE Runtime SHALL print a plain-language message identifying the port conflict and exit with a non-zero exit code

---

### Requirement 11: Manifest Schema Versioning

**User Story:** As a developer, I want the manifest format to be versioned, so that future schema changes do not silently break existing Runtime executables.

#### Acceptance Criteria

1. THE Manifest SHALL include a `schema_version` integer field
2. WHEN the Runtime reads a manifest with a `schema_version` higher than it supports, THE Runtime SHALL log a warning and attempt to run using the fields it recognizes
3. WHEN the Runtime reads a manifest with a missing or zero `schema_version`, THE Runtime SHALL treat it as `schema_version: 1` and log a warning

---

### Requirement 12: Future Platform Support (Phase 2 — Out of Scope for MVP)

**User Story:** As a user on Linux, macOS, or with AMD/Intel hardware, I want the same experience to eventually be available on my platform, so that local AI is accessible regardless of my setup.

> **Note:** All criteria in this requirement are out of scope for the current phase. They are captured here for architectural planning only.

#### Acceptance Criteria

1. WHERE Linux is targeted in a future phase, THE Runtime SHALL support Ubuntu 22.04 and later on x86-64 with NVIDIA CUDA and AMD Vulkan backends
2. WHERE macOS is targeted in a future phase, THE Runtime SHALL support macOS 13 and later on Apple Silicon using the Metal backend and be distributed as a `.app` bundle
3. WHERE AMD/Intel GPU support is targeted in a future phase, THE Runtime SHALL use the Vulkan backend on Windows and Linux
4. THE Packager architecture SHALL be designed so that adding a new platform backend does not require rewriting the hardware detection or configuration logic
