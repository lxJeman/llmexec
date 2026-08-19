# Implementation Plan: Local AI Runtime

## Overview

Build the Local AI Runtime in two parts: a **Packager CLI** (developer tool) and a **Runtime executable** (end-user app). The implementation language is Rust, which naturally produces single-file Windows executables, has excellent C FFI for llama.cpp, and has `proptest` for property-based testing.

Implementation progresses bottom-up: data models and validation first, then hardware detection, then configuration logic, then the inference engine wrapper, then the chat interface, and finally the packager CLI that bundles everything.

---

## Tasks

- [ ] 1. Project scaffold and tooling setup
  - Initialize a Cargo workspace with two crates: `packager` (bin) and `runtime` (bin), and a shared `core` lib crate
  - Add dependencies: `llama_cpp` or raw `llama-sys` bindings, `serde`/`serde_json`, `proptest`, `clap`, `windows-sys` (for `GlobalMemoryStatusEx`), `log`/`env_logger`
  - Set up a `build.rs` that links the pre-built llama.cpp static or shared library
  - Configure `Cargo.toml` workspace and verify `cargo build` succeeds for all crates
  - _Requirements: 1.2, 5.1_

- [ ] 2. GGUF validation and manifest data models
  - [ ] 2.1 Implement GGUF validator in `core`
    - Parse the 4-byte magic (`GGUF`), version field (accept 2 or 3), tensor count, and metadata KV count from a file or byte slice
    - Return a typed `GgufHeader` on success or a descriptive `ValidationError` on failure
    - _Requirements: 5.4, 5.5_
  - [ ]* 2.2 Write property tests for GGUF validator (Property 6 & 7)
    - Generate random byte sequences without valid magic/version → must return error
    - Generate well-formed GGUF headers (magic + version 2 or 3 + non-zero counts) → must return success
    - `// Feature: local-ai-runtime, Property 6 & 7: GGUF validation rejects/accepts files`
    - _Requirements: 5.4, 5.5_
  - [ ] 2.3 Implement `Manifest` struct and JSON serialization in `core`
    - Fields: `name: String`, `model_file: String`, `default_context_size: Option<u32>`, `version: u32`
    - Implement `serialize` → `serde_json::to_string` and `deserialize` → `serde_json::from_str`
    - Missing fields must fall back to safe defaults; extra fields must be ignored
    - _Requirements: 5.2, 5.3_
  - [ ]* 2.4 Write property test for manifest round trip (Property 8)
    - Generate random valid `Manifest` objects; serialize then deserialize; assert structural equality
    - `// Feature: local-ai-runtime, Property 8: Manifest round trip`
    - _Requirements: 5.3_

- [ ] 3. Checkpoint — core data layer
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 4. Hardware detection
  - [ ] 4.1 Implement `HardwareDetector` in `core`
    - Dynamically load `nvml.dll` via `windows-sys` `LoadLibraryA`; if unavailable set `has_nvidia_gpu = false`
    - When NVML loads: call `nvmlInit`, `nvmlDeviceGetCount`, `nvmlDeviceGetHandleByIndex(0)`, `nvmlDeviceGetMemoryInfo` to populate `gpu_name` and `gpu_vram_bytes`
    - Query `GlobalMemoryStatusEx` for `system_ram_bytes`
    - Return `HardwareInfo` struct; any NVML query failure must set `has_nvidia_gpu = false` and continue
    - Write results to the structured log
    - _Requirements: 2.1, 2.2, 2.3, 2.4_
  - [ ]* 4.2 Write unit tests for hardware detector
    - Test NVML-unavailable path (mock/stub the dynamic load): must produce `has_nvidia_gpu = false`
    - Test NVML-available but device-query-failure path: must also produce `has_nvidia_gpu = false`
    - _Requirements: 2.3_

- [ ] 5. Backend selection and model configurator
  - [ ] 5.1 Implement `Configurator` in `core`
    - Select backend: CUDA when `has_nvidia_gpu == true`, CPU otherwise; log the decision and reason
    - Calculate `gpu_layers`: `floor((vram_bytes - 512MB_buffer) / bytes_per_layer)`, clamped to `[0, total_layers]`
    - Select context size: choose the largest value from `[512, 1024, 2048, 4096, 8192, 16384, 32768]` that fits remaining memory; respect manifest `default_context_size` as an upper cap
    - Return `InferenceConfig { backend, gpu_layers, context_size, model_path }`
    - _Requirements: 3.1, 3.2, 3.3, 4.1, 4.2, 4.4_
  - [ ]* 5.2 Write property tests for Configurator (Properties 1, 2, 3, 4, 5)
    - **Property 1 & 2**: Generate random VRAM sizes (0–80 GB) and layer configs; assert `gpu_layers * bytes_per_layer + 512MB <= vram` and `gpu_layers <= total_layers`
    - **Property 3 & 4**: After layer assignment, assert context size is in the standard set and fits remaining memory
    - **Property 5**: For any config with `has_nvidia_gpu = false`, assert backend == CPU and `gpu_layers == 0`
    - `// Feature: local-ai-runtime, Property 1-5: Configurator correctness`
    - _Requirements: 3.2, 4.1, 4.2_
  - [ ] 5.3 Implement CUDA fallback on initialization failure
    - If `llama_backend_init` or context creation returns an error when backend == CUDA, catch it, log the error, switch backend to CPU, set `gpu_layers = 0`, notify the user in plain language, and retry
    - _Requirements: 3.4, 7.2_
  - [ ]* 5.4 Write property test for CUDA fallback (Property 9)
    - Simulate CUDA init failure (stub the init call to return an error); assert resulting `InferenceConfig` has `backend == CPU` and `gpu_layers == 0`
    - `// Feature: local-ai-runtime, Property 9: CUDA fallback on initialization failure`
    - _Requirements: 3.4_

- [ ] 6. Checkpoint — configuration logic
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 7. Inference engine wrapper
  - [ ] 7.1 Implement `InferenceEngine` in `runtime` crate
    - `load_model(config: &InferenceConfig) -> Result<Model>` using `llama_model_load_from_file`
    - `create_context(model: &Model, context_size: u32) -> Result<Context>` using `llama_new_context_with_params`
    - `generate(context: &mut Context, prompt: &str, callback: impl Fn(&str)) -> Result<()>` — calls callback per token using `llama_batch_init`/`llama_decode` loop
    - `stop()` — sets an atomic flag that the decode loop checks after each token
    - Handle OOM from llama.cpp: surface as `Error::OutOfMemory` with a plain-language message
    - _Requirements: 6.2, 6.3, 6.5, 7.1_
  - [ ] 7.2 Wire Ctrl+C signal handler to `stop()`
    - Register a `ctrlc` or `windows-sys` console ctrl handler that calls `engine.stop()` when the user presses Ctrl+C
    - _Requirements: 6.5_
  - [ ]* 7.3 Write unit tests for inference engine error paths
    - Test that an OOM error from llama.cpp produces a `Error::OutOfMemory` with a user-readable message
    - Test that calling `stop()` before generation begins does not panic
    - _Requirements: 7.1_

- [ ] 8. Chat interface
  - [ ] 8.1 Implement terminal chat loop in `runtime/main.rs`
    - On startup: print hardware summary and `InferenceConfig` in plain language (e.g., "Running on RTX 4060 — 8 GB GPU memory, all 32 layers on GPU")
    - Loop: print `> ` prompt, read a line from stdin, append to conversation history as `{role: user, content}`
    - Build the full prompt using the model's chat template from GGUF metadata (fall back to ChatML format if absent)
    - Call `engine.generate()` with a callback that writes each token to stdout immediately (no buffering)
    - After generation: print newline, print `tokens/sec` (first-token timestamp to last-token timestamp), append response to history
    - On clean exit (empty input or EOF): call `engine.stop()`, free resources, exit 0
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.6, 6.7_
  - [ ] 8.2 Implement structured logger
    - Write timestamped log lines to `<exe_dir>/runtime.log` (fall back to `%TEMP%/localai_<name>/runtime.log`)
    - Log: hardware summary, backend selection, config, model load time, any errors
    - On unrecoverable error: display log file path and a brief message before exiting (no silent crash)
    - _Requirements: 7.3, 7.4_

- [ ] 9. Command-line flag overrides
  - Add `--gpu-layers <n>`, `--context <n>`, and `--backend cpu|cuda` flags to the Runtime binary via `clap`
  - When provided, these override the Configurator's calculated values in `InferenceConfig`
  - _Requirements: 4.4_

- [ ] 10. Checkpoint — runtime executable
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 11. Packager CLI
  - [ ] 11.1 Implement Packager binary in `packager` crate
    - Accept `--model <path>`, `--name <string>`, `--context <int>` (optional), `--output <path>` (default `<name>.exe`) via `clap`
    - Validate the GGUF file using the `core` validator; print a descriptive error and exit non-zero if invalid
    - Write `manifest.json` with the provided name, model filename, and optional default context size
    - Copy the GGUF file and pre-built Runtime host binary + DLLs (`cublas64_12.dll`, `cudart64_12.dll`, `llama.dll`) into the output bundle directory
    - Produce the output `.exe` (self-extracting launcher or directory launcher)
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_
  - [ ]* 11.2 Write unit tests for Packager
    - Test that a valid GGUF produces a successful output bundle
    - Test that an invalid GGUF (wrong magic) prints an error and exits non-zero
    - Test that manifest fields round-trip correctly through the bundle
    - _Requirements: 5.4, 5.5_

- [ ] 12. Self-extracting launcher (single-file distribution)
  - Implement the startup extraction step in the Runtime: at launch, detect if bundled DLLs are embedded as resources; if so, extract them to a temp directory and load from there before any llama.cpp calls
  - This makes `MyModel.exe` a true single file rather than a directory bundle
  - _Requirements: 1.1, 1.2, 1.3_

- [ ] 13. Final checkpoint — end-to-end
  - Ensure all tests pass, ask the user if questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- Property tests each reference the numbered property in `design.md` and run ≥ 100 iterations
- The Rust `proptest` crate is used for all property-based tests
- Tag format: `// Feature: local-ai-runtime, Property N: <title>`
