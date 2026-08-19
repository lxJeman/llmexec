# Requirements Document

## Introduction

A tool that packages a specific LLM (in GGUF format), llama.cpp, and the CUDA runtime into a distributable Windows application that end users can download and double-click to run. The goal is to eliminate all setup friction for non-technical users: no Python, no CUDA toolkit installation, no terminal, no configuration. The developer uses this tool to produce the distributable; the end user simply runs it.

The distribution consists of two executables: a **Setup executable** that handles the one-time runtime installation, and a **Model executable** that runs the model. On first launch the Setup checks whether the shared runtime components are already present on the system. If not, the user chooses between a system-wide install (shared across all model executables) or a portable install (runtime extracted next to the exe). Subsequent launches skip setup entirely.

The developer also chooses at package time whether the model executable presents a terminal interface or a simple GUI, depending on the intended use of the model.

The initial target is Windows with NVIDIA GPU (CUDA), with Linux, macOS, AMD (Vulkan), and Apple Silicon (Metal) support added in later phases.

## Glossary

- **Packager**: The developer-facing tool that bundles a model, llama.cpp, and the CUDA runtime into a distributable executable
- **Runtime**: The self-contained executable produced by the Packager that the end user downloads and runs
- **Backend**: The GPU/CPU compute library used for inference (CUDA for NVIDIA, CPU as fallback for MVP)
- **Model**: A GGUF-format language model file used for inference
- **Inference Engine**: The embedded llama.cpp instance used for token generation
- **GGUF**: The model file format used by llama.cpp
- **Quantization**: A compression technique that reduces model size and memory usage (e.g., Q4_K_M, Q8_0)
- **GPU Layers**: The number of model layers offloaded to GPU VRAM for acceleration
- **Context Size**: The maximum number of tokens the model can process in one session
- **Hardware Detector**: The Runtime subsystem that identifies the user's GPU, VRAM, and available system RAM at startup
- **Configurator**: The Runtime subsystem that selects GPU layer count and context size based on detected hardware

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
3. THE Configurator SHALL display the chosen configuration to the user in plain language before inference begins (e.g., "Running on RTX 4060 — 8 GB GPU memory, all layers on GPU")
4. WHERE an advanced user wants manual control, THE Runtime SHALL accept optional command-line flags to override GPU layer count, context size, and backend

---

### Requirement 5: Model Bundling by the Developer

**User Story:** As a developer, I want to bundle a specific GGUF model into the Runtime executable, so that my end users receive a single file that contains everything needed to run that model.

#### Acceptance Criteria

1. THE Packager SHALL accept a GGUF model file as input and produce a single `.exe` that contains the model, llama.cpp, and the CUDA runtime
2. THE Packager SHALL accept configuration inputs including the model file path, a display name, and optional default context size
3. WHEN the Runtime is built, THE Packager SHALL embed the GGUF model data into the executable or bundle it as a required sidecar file in the same directory
4. THE Packager SHALL validate that the provided model file is a valid GGUF before producing the output executable
5. IF the provided model file is not a valid GGUF, THEN THE Packager SHALL report a descriptive error and exit without producing output

---

### Requirement 6: Chat Interface

**User Story:** As a user, I want a simple conversational interface when I launch the executable, so that I can interact with the model without using a terminal directly.

#### Acceptance Criteria

1. THE Runtime SHALL provide a text-based or graphical chat interface for sending messages and receiving model responses
2. WHEN the user sends a message, THE Runtime SHALL begin streaming the response token by token without waiting for the full response
3. WHILE a response is streaming, THE Runtime SHALL display tokens as they arrive
4. THE Runtime SHALL display the current inference speed in tokens per second during generation
5. THE Runtime SHALL allow the user to stop generation at any point
6. THE Runtime SHALL maintain conversation history within a session
7. WHEN the user exits, THE Runtime SHALL terminate cleanly without leaving background processes

---

### Requirement 7: Error Handling and User Feedback

**User Story:** As a non-technical user, I want clear, plain-language feedback when something goes wrong, so that I understand what happened and what I can do.

#### Acceptance Criteria

1. IF model loading fails due to insufficient memory, THEN THE Runtime SHALL display a plain-language message explaining the issue and suggest reducing GPU layers or switching to CPU
2. IF the CUDA backend fails to initialize, THEN THE Runtime SHALL display which backend was attempted, why it failed, and confirm the fallback being used
3. THE Runtime SHALL write a structured log file to a known location (e.g., next to the executable or in a temp folder) for diagnostic purposes
4. IF the Runtime encounters an unrecoverable error, THEN THE Runtime SHALL display the log file location and a brief message before exiting rather than silently crashing

---

### Requirement 8: Future Platform Support (Phase 2)

**User Story:** As a user on Linux, macOS, or with AMD/Intel hardware, I want the same experience to eventually be available on my platform, so that local AI is accessible regardless of my setup.

#### Acceptance Criteria

1. WHERE Linux is targeted in a future phase, THE Runtime SHALL support Ubuntu 22.04 and later on x86-64 with NVIDIA CUDA and AMD Vulkan backends
2. WHERE macOS is targeted in a future phase, THE Runtime SHALL support macOS 13 and later on Apple Silicon using the Metal backend and be distributed as a `.app` bundle
3. WHERE AMD/Intel GPU support is targeted in a future phase, THE Runtime SHALL use the Vulkan backend on Windows and Linux
4. THE Packager architecture SHALL be designed so that adding a new platform backend does not require rewriting the hardware detection or configuration logic
