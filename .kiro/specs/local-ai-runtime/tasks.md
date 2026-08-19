# Implementation Plan: Local AI Runtime

## Overview

Build the Local AI Runtime in Rust: a `core` lib crate (shared logic), a `runtime` bin crate (the end-user executable), and a `packager` bin crate (developer CLI). Tasks progress bottom-up — data models → hardware detection → configuration → inference engine → API server → web UI serving → chat mode → packager.

---

## Tasks

- [ ] 1. Project scaffold and tooling setup
  - Initialize a Cargo workspace with three crates: `core` (lib), `runtime` (bin), `packager` (bin)
  - Add dependencies: `llama-sys` or `llama_cpp` bindings, `serde`/`serde_json`, `proptest`, `clap`, `axum`, `tokio`, `tower-http`, `windows-sys`, `log`/`env_logger`, `ctrlc`
  - Set up `build.rs` in `core` that links the pre-built llama.cpp static library and bundles CUDA DLLs
  - Verify `cargo build --workspace` succeeds
  - _Requirements: 1.2, 5.1_

- [ ] 2. GGUF validation and Manifest data models
  - [ ] 2.1 Implement GGUF validator in `core`
    - Parse 4-byte magic (`GGUF`), version field (accept 2 or 3), tensor count, and metadata KV count from a byte slice or file
    - Return a typed `GgufHeader` on success or a descriptive `ValidationError` on failure
    - _Requirements: 5.4, 5.5_
  - [ ]* 2.2 Write property tests for GGUF validator (Properties 6 & 7)
    - Generate random byte sequences without valid magic/version → must return error
    - Generate well-formed GGUF headers (correct magic + version 2 or 3 + non-zero counts) → must return success
    - `// Feature: local-ai-runtime, Property 6 & 7: GGUF validation rejects/accepts files`
    - _Requirements: 5.4, 5.5_
  - [ ] 2.3 Implement `Manifest` struct and JSON serialization in `core`
    - Fields: `name`, `model_file`, `default_mode` (server/chat), `default_context_size: Option<u32>`, `api_key_hash: Option<String>`, `version: u32`
    - `deserialize`: missing fields fall back to safe defaults; extra fields are ignored
    - _Requirements: 5.2, 5.3_
  - [ ]* 2.4 Write property test for manifest round trip (Property 8)
    - Generate random valid `Manifest` objects; serialize then deserialize; assert structural equality
    - `// Feature: local-ai-runtime, Property 8: Manifest round trip`
    - _Requirements: 5.2, 5.3_

- [ ] 3. Checkpoint — data layer
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 4. Hardware detection
  - [ ] 4.1 Implement `HardwareDetector` in `core`
    - Dynamically load `nvml.dll` via `windows-sys`; on failure set `has_nvidia_gpu = false` and continue
    - When NVML loads: call `nvmlInit`, query device 0 for `gpu_name` and `gpu_vram_bytes`
    - Query `GlobalMemoryStatusEx` for `system_ram_bytes`
    - Return `HardwareInfo`; any NVML query failure must fall back to CPU mode
    - Write a summary line to the structured log
    - _Requirements: 2.1, 2.2, 2.3, 2.4_
  - [ ]* 4.2 Write unit tests for hardware detector
    - Stub the dynamic load to simulate NVML unavailable → `has_nvidia_gpu = false`
    - Stub NVML loaded but device query failure → also `has_nvidia_gpu = false`
    - _Requirements: 2.3_

- [ ] 5. Configurator
  - [ ] 5.1 Implement `Configurator` in `core`
    - Backend selection: CUDA when `has_nvidia_gpu == true`, CPU otherwise; log decision
    - GPU layer calc: `floor((vram_bytes - 512MB) / bytes_per_layer)` clamped to `[0, total_layers]`
    - Context size: largest from `[512, 1024, 2048, 4096, 8192, 16384, 32768]` that fits remaining memory; respect `manifest.default_context_size` as upper cap
    - Return `InferenceConfig { backend, gpu_layers, context_size, model_path }`
    - _Requirements: 3.1, 3.2, 3.3, 4.1, 4.2_
  - [ ]* 5.2 Write property tests for Configurator (Properties 1, 2, 3, 4, 5)
    - **Property 1 & 2**: Random VRAM sizes (0–80 GB) and layer configs; assert `gpu_layers * bytes_per_layer + 512MB <= vram` and `gpu_layers <= total_layers`
    - **Property 3 & 4**: After layer assignment, assert context size is in the standard set and fits remaining memory
    - **Property 5**: For any config with `has_nvidia_gpu = false`, assert `backend == CPU` and `gpu_layers == 0`
    - `// Feature: local-ai-runtime, Property 1-5: Configurator correctness`
    - _Requirements: 3.2, 4.1, 4.2_
  - [ ] 5.3 Implement CUDA fallback on initialization failure in `core`
    - If `llama_backend_init` or context creation fails when backend == CUDA: log the error, switch to CPU, set `gpu_layers = 0`, surface a user-readable message
    - _Requirements: 3.4, 9.2_
  - [ ]* 5.4 Write property test for CUDA fallback (Property 9)
    - Stub CUDA init to return an error; assert resulting `InferenceConfig` has `backend == CPU` and `gpu_layers == 0`
    - `// Feature: local-ai-runtime, Property 9: CUDA fallback on initialization failure`
    - _Requirements: 3.4_

- [ ] 6. Checkpoint — configuration logic
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 7. Inference engine wrapper
  - [ ] 7.1 Implement `InferenceEngine` in `core`
    - `load_model(config: &InferenceConfig) -> Result<Model>` via `llama_model_load_from_file`
    - `create_context(model: &Model, context_size: u32) -> Result<Context>` via `llama_new_context_with_params`
    - `generate(context: &mut Context, prompt: &str, callback: impl Fn(&str)) -> Result<GenerationStats>` — calls callback per token, returns `{ tokens_generated, elapsed_ms, tokens_per_sec }`
    - `stop()` — atomic flag; inference loop checks it after each decoded token
    - OOM from llama.cpp → `Error::OutOfMemory` with a plain-language message
    - _Requirements: 6.2 (via API), 9.1_
  - [ ] 7.2 Build chat template formatter in `core`
    - Read the chat template from GGUF metadata if present; fall back to ChatML format
    - `format_messages(messages: &[Message], template: &Template) -> String`
    - _Requirements: 6.2 (via API)_
  - [ ]* 7.3 Write unit tests for inference engine error paths
    - OOM error → `Error::OutOfMemory` with readable message
    - Calling `stop()` before generation starts does not panic
    - _Requirements: 9.1_

- [ ] 8. API Server — core endpoints
  - [ ] 8.1 Implement the axum HTTP server in `runtime`
    - Bind to `host:port` from `InferenceConfig` (default `127.0.0.1:8080`)
    - Share `InferenceEngine` behind an `Arc<Mutex<>>` or request queue channel
    - Implement `GET /health` → `{"status":"ok","model_loaded":true}`
    - Implement `GET /v1/models` → OpenAI models list with the bundled model name
    - Implement `GET /metrics` → `{"tokens_per_sec": N, "active_requests": N, "total_requests": N}`
    - If port is already in use: log plain-language error and exit
    - _Requirements: 6.4, 6.5, 6.6, 8.4, 8.5, 9.5_
  - [ ]* 8.2 Write unit tests for health and models endpoints
    - `GET /health` returns 200 after model load
    - `GET /v1/models` returns the correct model name
    - _Requirements: 6.4, 6.5_

- [ ] 9. API Server — chat completions
  - [ ] 9.1 Implement `POST /v1/chat/completions` (non-streaming)
    - Parse OpenAI-compatible request body (`model`, `messages`, `max_tokens`, `temperature`)
    - Format messages via chat template, call `engine.generate()`
    - Return OpenAI-compatible response with `id`, `object`, `created`, `model`, `choices`, `usage`
    - _Requirements: 6.2_
  - [ ] 9.2 Implement streaming via Server-Sent Events
    - When request contains `"stream": true`: return SSE response
    - Each token callback emits `data: <delta json>\n\n`
    - Terminate stream with `data: [DONE]\n\n`
    - _Requirements: 6.3_
  - [ ] 9.3 Implement request queuing
    - Concurrent POST requests are queued and processed one at a time
    - Waiting clients receive their response when their turn arrives (no 429)
    - _Requirements: 6.7_
  - [ ] 9.4 Implement optional API key authentication
    - If `manifest.api_key_hash` is set, require `Authorization: Bearer <key>` on all `/v1/` requests
    - `/health` and `/metrics` remain unauthenticated
    - _Requirements: 6.8_
  - [ ]* 9.5 Write property tests for API response correctness (Properties 10, 11, 12)
    - **Property 10**: Random valid request inputs; assert response contains all required OpenAI fields
    - **Property 11**: Random streaming requests; assert SSE stream ends with `data: [DONE]`
    - **Property 12**: Post-load state; assert `GET /health` returns HTTP 200
    - `// Feature: local-ai-runtime, Property 10-12: API correctness`
    - _Requirements: 6.2, 6.3, 6.5_

- [ ] 10. Checkpoint — API server
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 11. Web UI serving and chat mode
  - [ ] 11.1 Add static file serving to the API server
    - `GET /*` serves files from the bundled `web/` asset directory
    - If no web assets are bundled, return 404 for all non-API routes
    - _Requirements: 7.1, 7.7_
  - [ ] 11.2 Implement chat mode startup behavior in `runtime/main.rs`
    - After the API server is listening, if mode == chat and web assets are bundled: open `http://localhost:<port>` in the default browser via `std::process::Command` (Windows: `start`, Linux: `xdg-open`, macOS: `open`)
    - If mode == chat but no web assets bundled: log warning "No UI assets bundled, running in server-only mode"
    - _Requirements: 7.1, 7.7_
  - [ ] 11.3 Build the Web UI (HTML/CSS/JS single-page app)
    - Chat input field and submit button
    - Conversation history panel with rendered text
    - Configuration summary bar: model name, backend, GPU layers, context size (fetched from `/health` or embedded in page)
    - Tokens/sec display during generation (parsed from streaming response)
    - Stop generation button (aborts the fetch)
    - Communicates with `/v1/chat/completions` using the Fetch API with streaming (`ReadableStream`)
    - _Requirements: 7.2, 7.3, 7.4, 7.5, 7.6_

- [ ] 12. CLI flags and mode selection
  - Add to `runtime` binary via `clap`: `--chat`, `--server`, `--port <n>`, `--host <addr>`, `--gpu-layers <n>`, `--context <n>`, `--backend cpu|cuda`
  - Mode flags override the packaged manifest default
  - Hardware override flags feed into the Configurator after its automatic calculation
  - _Requirements: 4.4, 8.1, 8.2, 8.3, 8.4, 8.5_

- [ ] 13. Structured logger and error display
  - Write timestamped log lines to `<exe_dir>/runtime.log` (fall back to `%TEMP%/localai_<name>/runtime.log`)
  - Log: hardware summary, backend selection, config, model load time, each request (method + path + status), errors
  - On unrecoverable error: print log file path and a brief plain-language message before exit (no silent crash)
  - _Requirements: 9.3, 9.4_

- [ ] 14. Checkpoint — runtime executable
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 15. Packager CLI
  - [ ] 15.1 Implement `packager` binary
    - Accept `--model`, `--name`, `--mode`, `--context`, `--ui-assets`, `--api-key`, `--output` via `clap`
    - Validate GGUF via `core` validator; descriptive error + non-zero exit on failure
    - If `--mode chat` and `--ui-assets` not provided: error and exit
    - Write `manifest.json`; copy GGUF + runtime host binary + DLLs + web assets into output bundle
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_
  - [ ]* 15.2 Write unit tests for Packager
    - Valid GGUF → successful output bundle with correct manifest fields
    - Invalid GGUF (wrong magic) → error message + non-zero exit
    - `--mode chat` without `--ui-assets` → error message + non-zero exit
    - _Requirements: 5.4, 5.5_

- [ ] 16. Self-extracting single-file launcher
  - At Runtime startup: detect if bundled DLLs are embedded as resource sections in the `.exe`
  - If embedded: extract to `%TEMP%/localai_<name>/` and load from there before any llama.cpp calls
  - This makes `MyModel.exe` a true single file rather than a directory bundle
  - _Requirements: 1.1, 1.2, 1.3_

- [ ] 17. Final checkpoint — end-to-end
  - Ensure all tests pass, ask the user if questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- Property tests each reference the numbered property in `design.md` and run ≥ 100 iterations via `proptest`
- Tag format: `// Feature: local-ai-runtime, Property N: <title>`
