# Requirements Document

## Introduction

A tool that packages a specific LLM (in GGUF format), llama.cpp, and the CUDA runtime into a distributable Windows application. The developer uses the **Packager** CLI to produce a self-contained `.exe`; end users download that `.exe` and run it with no prior setup.

The Runtime supports two operating modes chosen at launch via a flag:

- **Server mode** (default): runs headless and exposes an OpenAI-compatible HTTP API on the local network, allowing any OpenAI-compatible client to connect
- **Chat mode**: additionally opens a browser-based chat UI served from the same process

The developer may optionally bundle a pre-built web UI into the `.exe` at package time. When the Runtime launches in chat mode, it opens the user's default browser to the local UI automatically.

The initial target is Windows with NVIDIA GPU (CUDA), with Linux, macOS, AMD (Vulkan), and Apple Silicon (Metal) support added in later phases.

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
- **Context Size**: The maximum number of tokens the model can process in one session
- **Hardware Detector**: The Runtime subsystem that identifies the user's GPU, VRAM, and available system RAM at startup
- **Configurator**: The Runtime subsystem that selects GPU layer count and context size based on detected hardware
- **API Server**: The Runtime subsystem that handles HTTP requests and serves the OpenAI-compatible REST API

---

## Requirements

### Requirement 1: Single-Executable Distribution

**User Story:** As a non-technical user, I want to download and run a single file without any prior setup, so that I can use a local AI model without installing Python, CUDA, or any other dependency.

#### Acceptance Criteria

1. THE Runtime SHALL be distributed as a single `.exe` file on Windows
2. THE Runtime SHALL bundle all required libraries including the CUDA runtime, cuBLAS, and llama.cpp within the executable package
3. WHEN the Runtime is launched, THE Runtime SHALL start without requiring the user to install any additional software
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
4. IF the CUDA backend fails to initialize at runtime, THEN THE Configurator SHALL automatically fall back to the CPU backend and notify the user

---

### Requirement 4: Automatic Model Configuration

**User Story:** As a user, I want the runtime to automatically determine how to load the model given my hardware, so that I don't have to understand GPU layers, quantization, or context sizes.

#### Acceptance Criteria

1. WHEN the Runtime starts, THE Configurator SHALL calculate the maximum number of GPU layers that fit within available VRAM
2. WHEN the Runtime starts, THE Configurator SHALL select a context size that fits within remaining memory after model layers are allocated
3. THE Configurator SHALL log the chosen configuration in plain language before inference begins (e.g., "Running on RTX 4060 — 8 GB GPU memory, all layers on GPU")
4. WHERE an advanced user wants manual control, THE Runtime SHALL accept optional command-line flags to override GPU layer count, context size, and backend

---

### Requirement 5: Model Bundling by the Developer

**User Story:** As a developer, I want to bundle a specific GGUF model into the Runtime executable, so that my end users receive a single file that contains everything needed to run that model.

#### Acceptance Criteria

1. THE Packager SHALL accept a GGUF model file as input and produce a single `.exe` that contains the model, llama.cpp, and the CUDA runtime
2. THE Packager SHALL accept configuration inputs including the model file path, a display name, default operating mode (`server` or `chat`), optional default context size, and optional web UI assets directory
3. WHEN the Runtime is built, THE Packager SHALL embed the GGUF model data into the executable or bundle it as a required sidecar file in the same directory
4. THE Packager SHALL validate that the provided model file is a valid GGUF before producing the output executable
5. IF the provided model file is not a valid GGUF, THEN THE Packager SHALL report a descriptive error and exit without producing output

---

### Requirement 6: Server Mode — OpenAI-Compatible HTTP API

**User Story:** As a developer or power user, I want the runtime to expose an OpenAI-compatible HTTP API on the local network, so that I can connect any existing OpenAI-compatible client or tool to my local model without any custom integration work.

#### Acceptance Criteria

1. WHEN the Runtime starts in server mode, THE API Server SHALL listen on a configurable port (default 8080) on localhost and optionally on all network interfaces
2. THE API Server SHALL implement `POST /v1/chat/completions` accepting an OpenAI-compatible request body and returning an OpenAI-compatible response
3. WHEN a client requests streaming via `"stream": true`, THE API Server SHALL return a Server-Sent Events stream of incremental tokens in OpenAI delta format
4. THE API Server SHALL implement `GET /v1/models` returning the bundled model's name and metadata
5. THE API Server SHALL implement `GET /health` returning HTTP 200 with a JSON body indicating readiness
6. THE API Server SHALL implement `GET /metrics` returning current inference statistics including tokens per second and active request count
7. WHEN multiple requests arrive concurrently, THE API Server SHALL queue them and process one at a time, returning an appropriate status to waiting clients
8. WHERE the developer configures an API key at package time, THE API Server SHALL require that key as a Bearer token on all `/v1/` endpoints

---

### Requirement 7: Chat Mode — Browser UI

**User Story:** As a non-technical user, I want to open a browser and interact with the model through a chat interface, so that I can use it without understanding HTTP APIs or terminals.

#### Acceptance Criteria

1. WHEN the Runtime is launched with the `--chat` flag or was packaged with `--mode chat`, THE Runtime SHALL start the API Server and then open the user's default browser to `http://localhost:<port>`
2. THE Web UI SHALL display a chat input, a conversation history panel, and a configuration summary (model name, backend, GPU layers, context size)
3. WHEN the user submits a message, THE Web UI SHALL stream the model's response token by token as it arrives
4. THE Web UI SHALL display tokens-per-second during active generation
5. THE Web UI SHALL allow the user to stop generation
6. THE Web UI SHALL maintain conversation history for the current session
7. WHERE the developer did not bundle web UI assets, THEN THE Runtime SHALL log a warning and fall back to server-only mode rather than opening a broken browser page

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

### Requirement 9: Error Handling and User Feedback

**User Story:** As a non-technical user, I want clear, plain-language feedback when something goes wrong, so that I understand what happened and what I can do.

#### Acceptance Criteria

1. IF model loading fails due to insufficient memory, THEN THE Runtime SHALL display a plain-language message explaining the issue and suggest reducing GPU layers or switching to CPU
2. IF the CUDA backend fails to initialize, THEN THE Runtime SHALL display which backend was attempted, why it failed, and confirm the fallback being used
3. THE Runtime SHALL write a structured log file to a known location (next to the executable or in `%TEMP%/localai_<name>/`) for diagnostic purposes
4. IF the Runtime encounters an unrecoverable error, THEN THE Runtime SHALL display the log file location and a brief message before exiting rather than silently crashing
5. IF the requested port is already in use, THEN THE Runtime SHALL report the port conflict and the port number in plain language and exit

---

### Requirement 10: Future Platform Support (Phase 2)

**User Story:** As a user on Linux, macOS, or with AMD/Intel hardware, I want the same experience to eventually be available on my platform, so that local AI is accessible regardless of my setup.

#### Acceptance Criteria

1. WHERE Linux is targeted in a future phase, THE Runtime SHALL support Ubuntu 22.04 and later on x86-64 with NVIDIA CUDA and AMD Vulkan backends
2. WHERE macOS is targeted in a future phase, THE Runtime SHALL support macOS 13 and later on Apple Silicon using the Metal backend and be distributed as a `.app` bundle
3. WHERE AMD/Intel GPU support is targeted in a future phase, THE Runtime SHALL use the Vulkan backend on Windows and Linux
4. THE Packager architecture SHALL be designed so that adding a new platform backend does not require rewriting the hardware detection or configuration logic
