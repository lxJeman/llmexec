# Implementation Plan: Local AI Runtime

## Overview

Build the Local AI Runtime in Rust: a `core` lib crate (shared logic), a `runtime` bin crate (the end-user executable), and a `packager` bin crate (developer CLI). Tasks progress bottom-up: data models and self-extractor first (because they are load-bearing infrastructure), then hardware detection, configuration, inference engine, API server, web UI, packager, and finally end-to-end wiring.

The chosen llama.cpp crate is `llama-cpp-rs`. The chosen property-based testing library is `proptest`.

---

## Tasks

- [ ] 1. Project scaffold and tooling setup
  - Initialize a Cargo workspace with three crates: `core` (lib), `runtime` (bin), `packager` (bin)
  - Add dependencies: `llama_cpp` (llama-cpp-rs), `serde`/`serde_json`, `proptest`, `clap`, `axum`, `tokio`, `tower-http`, `windows-sys`, `log`/`env_logger`, `ctrlc`, `subtle`, `uuid`
  - Set up `build.rs` in `core` that compiles llama.cpp via `llama-cpp-rs`'s build system
  - Verify `cargo build --workspace` succeeds
  - _Requirements: 1.2, 5.1_

- [ ] 2. Self-Extractor
  - [ ] 2.1 Implement the Self-Extractor in `core`
    - Compute extraction dir: `%TEMP%\localai_<name>\<sha256_of_exe_first_64kb>\`
    - Check for `extracted.ok` sentinel; skip extraction if present (fast path)
    - Read each DLL from the `.exe` Win32 resource section via `FindResource` / `LoadResource` / `LockResource` (`windows-sys`)
    - Write DLLs to disk, then write `extracted.ok`
    - Call `SetDllDirectoryW` to prepend the extraction dir to the DLL search path
    - _Requirements: 1.1, 1.2, 1.3_
  - [ ]* 2.2 Write unit tests for Self-Extractor
    - Verify sentinel fast-path skips extraction when `extracted.ok` exists
    - Verify `SetDllDirectoryW` is called with the correct path
    - _Requirements: 1.3_

- [ ] 3. GGUF validation and Manifest data models
  - [ ] 3.1 Implement GGUF validator in `core`
    - Parse 4-byte magic at offset 0, version u32 at offset 4 (accept 2 or 3), tensor count u64 at offset 8, metadata KV count u64 at offset 16
    - Return `GgufHeader` on success or descriptive `ValidationError` on failure
    - _Requirements: 5.4, 5.5_
  - [ ]* 3.2 Write property tests for GGUF validator (Property 4)
    - Generate random byte sequences without valid magic/version → must error
    - Generate well-formed headers (correct magic + version 2 or 3 + non-zero counts) → must succeed
    - `// Feature: local-ai-runtime, Property 4: GGUF validator correctly classifies all inputs`
    - _Requirements: 5.4, 5.5_
  - [ ] 3.3 Implement `Manifest` struct and JSON serialization in `core`
    - Fields: `name`, `model_file`, `default_mode`, `default_context_size: Option<u32>`, `api_key_hash: Option<String>`, `schema_version: u32`
    - Deserialization: missing fields fall back to safe defaults; unknown `schema_version > 1` logs warning; `schema_version` missing or 0 treated as 1 with warning
    - Extra fields are ignored (use `#[serde(deny_unknown_fields)]` opt-out)
    - _Requirements: 5.6, 11.1, 11.2, 11.3_
  - [ ]* 3.4 Write property test for Manifest round trip (Property 5)
    - Generate random valid `Manifest` objects; serialize then deserialize; assert structural equality
    - `// Feature: local-ai-runtime, Property 5: Manifest round trip`
    - _Requirements: 5.6_

- [ ] 4. Checkpoint — data layer and self-extractor
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 5. Hardware detection
  - [ ] 5.1 Implement `HardwareDetector` in `core`
    - Dynamically load `nvml.dll` via `LoadLibraryW`; on failure set `has_nvidia_gpu = false` and continue
    - When NVML loads: call `nvmlInit_v2`, query device 0 for `gpu_name` and `gpu_vram_bytes` via `nvmlDeviceGetMemoryInfo`
    - Query `GlobalMemoryStatusEx` for `system_ram_bytes`
    - Return `HardwareInfo`; any NVML query failure falls back to CPU mode
    - Write a summary line to the structured log
    - _Requirements: 2.1, 2.2, 2.3, 2.4_
  - [ ]* 5.2 Write unit tests for hardware detector
    - Stub dynamic load to simulate NVML unavailable → `has_nvidia_gpu = false`
    - Stub NVML loaded but device query failure → also `has_nvidia_gpu = false`
    - _Requirements: 2.3_

- [ ] 6. Configurator
  - [ ] 6.1 Implement `bytes_per_layer` derivation from GGUF tensor metadata in `core`
    - Read tensor descriptors from the GGUF file and sum attention + feed-forward tensor sizes for one layer
    - Fall back to 200 MB/layer if metadata is absent or malformed
    - _Requirements: 4.1_
  - [ ] 6.2 Implement `Configurator` in `core`
    - Backend: CUDA when `has_nvidia_gpu = true`, CPU otherwise; log decision with reason
    - GPU layer calc: `available = vram - 512MB`; if `<= 0` then `gpu_layers = 0`; else `min(floor(available / bytes_per_layer), total_layers)`
    - Context size: largest from `[512, 1024, 2048, 4096, 8192, 16384, 32768]` where `size * 512 <= remaining_memory`; cap at `manifest.default_context_size` if set
    - Return `InferenceConfig { backend, gpu_layers, context_size, model_path, bytes_per_layer }`
    - Write log line: "Backend: <X>, GPU layers: <n>/<total>, Context: <n> tokens, VRAM: <n> MB"
    - _Requirements: 3.1, 3.2, 3.3, 4.1, 4.2, 4.3_
  - [ ]* 6.3 Write property tests for Configurator (Properties 1, 2, 3)
    - **Property 1**: Random VRAM (0–80 GB), `bytes_per_layer` (50–500 MB), `total_layers` (1–128); assert `gpu_layers * bytes_per_layer + 512MB <= vram` AND `gpu_layers <= total_layers`
    - **Property 2**: After layer assignment with random configs, assert context size is in standard set and `context_size * 512 <= remaining_memory`
    - **Property 3**: Any `HardwareInfo` with `has_nvidia_gpu = false`; assert `backend == CPU` and `gpu_layers == 0`
    - `// Feature: local-ai-runtime, Property 1-3: Configurator correctness`
    - _Requirements: 3.2, 4.1, 4.2_
  - [ ] 6.4 Implement CUDA fallback on initialization failure in `core`
    - If llama.cpp backend init or context creation fails when `backend == CUDA`: log error, set `backend = CPU`, `gpu_layers = 0`, surface user-readable message (which backend failed and why)
    - _Requirements: 3.4, 10.2_
  - [ ]* 6.5 Write property test for CUDA fallback (Property 6)
    - Stub CUDA init to return error; assert resulting `InferenceConfig` has `backend == CPU` and `gpu_layers == 0`
    - `// Feature: local-ai-runtime, Property 6: CUDA init failure produces valid CPU config`
    - _Requirements: 3.4_

- [ ] 7. Checkpoint — configuration logic
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 8. Inference engine wrapper
  - [ ] 8.1 Implement `InferenceEngine` in `core`
    - `load_model(config: &InferenceConfig) -> Result<LlamaModel>` via `llama_cpp` crate
    - `create_session(model: &LlamaModel, context_size: u32) -> Result<LlamaSession>`
    - `generate(session: &mut LlamaSession, prompt: &str, stop_flag: Arc<AtomicBool>, callback: impl Fn(&str)) -> Result<GenerationStats>` — calls callback per token, checks `stop_flag` after each token
    - OOM from llama.cpp → `Error::OutOfMemory` with plain-language message
    - _Requirements: 6.2, 10.1_
  - [ ] 8.2 Implement context creation OOM retry in `core`
    - If `create_session` returns OOM: halve `context_size` and retry
    - If `context_size < 512` and still failing: return `Error::OutOfMemory`
    - _Requirements: 10.1_
  - [ ] 8.3 Implement chat template formatter in `core`
    - Read `tokenizer.chat_template` key from GGUF metadata; fall back to ChatML if absent
    - `format_messages(messages: &[Message], template: &ChatTemplate) -> String`
    - _Requirements: 6.2_
  - [ ]* 8.4 Write property test for chat template round trip (Property 10)
    - Generate random message lists; call `format_messages`; parse the output; assert role and content match for each message
    - `// Feature: local-ai-runtime, Property 10: Chat template format/parse round trip`
    - _Requirements: 6.2_
  - [ ]* 8.5 Write unit tests for inference engine error paths
    - OOM error → `Error::OutOfMemory` with readable message
    - OOM retry: context halving stops at minimum and surfaces error
    - `stop_flag` set before generation → generate returns without calling callback
    - _Requirements: 10.1_

- [ ] 9. API Server — core endpoints and request queue
  - [ ] 9.1 Implement the axum HTTP server skeleton in `runtime`
    - Bind to `host:port`; if port in use: log plain-language error and exit non-zero
    - Set up `tokio::sync::mpsc` inference queue channel shared via `axum` state
    - Spawn a `tokio::task::spawn_blocking` worker that reads from the channel and calls `generate()`; results returned via `oneshot::Sender`
    - Implement `GET /health` → `{"status":"ok","model_loaded":true}`
    - Implement `GET /v1/models` → OpenAI models list with bundled model name
    - Implement `GET /metrics` → `{"tokens_per_sec":N,"active_requests":N,"total_requests":N}`
    - _Requirements: 6.4, 6.5, 6.6, 8.4, 8.5, 10.5_
  - [ ]* 9.2 Write unit tests for health, models, and metrics endpoints
    - `GET /health` returns 200 after model load
    - `GET /v1/models` returns the correct model name
    - _Requirements: 6.4, 6.5_

- [ ] 10. API Server — chat completions, streaming, auth, and queuing
  - [ ] 10.1 Implement `POST /v1/chat/completions` non-streaming
    - Parse request body (`model`, `messages`, `stream`, `max_tokens`, `temperature`)
    - Format messages via chat template, enqueue to inference channel, await response
    - Return OpenAI response with `id`, `object`, `created`, `model`, `choices`, `usage`
    - _Requirements: 6.2_
  - [ ] 10.2 Implement streaming via SSE
    - When `stream: true`: return SSE response; emit `data: <delta json>\n\n` per token callback; terminate with `data: [DONE]\n\n`
    - _Requirements: 6.3_
  - [ ] 10.3 Implement optional API key authentication middleware
    - If `manifest.api_key_hash` is set: require `Authorization: Bearer <key>` on all `/v1/` routes
    - Compute `SHA-256(salt || provided_key)` and compare to stored digest using `subtle::ConstantTimeEq`
    - `/health` and `/metrics` are excluded from auth middleware
    - _Requirements: 6.8_
  - [ ]* 10.4 Write unit tests for CLI flag parsing
    - All flags (`--chat`, `--server`, `--port`, `--host`, `--gpu-layers`, `--context`, `--backend`) parse correctly
    - Override flags apply to `InferenceConfig` after Configurator runs
    - Default mode is server when no flag given
    - _Requirements: 4.4, 8.1, 8.2, 8.3, 8.4, 8.5_
  - [ ]* 10.5 Write unit tests for concurrent request queuing
    - Two simultaneous POST requests are queued; second receives its response after first completes
    - No 429 is returned; both requests get HTTP 200
    - _Requirements: 6.7_
  - [ ]* 10.6 Write property tests for API correctness (Properties 7, 8, 9)
    - **Property 7**: Random valid non-streaming requests; assert response has all required OpenAI fields
    - **Property 8**: Random streaming requests; assert every `data:` line before the last is valid JSON, last is `[DONE]`
    - **Property 9**: Post-load state; assert `GET /health` returns 200 with correct body
    - `// Feature: local-ai-runtime, Property 7-9: API response correctness`
    - _Requirements: 6.2, 6.3, 6.5_

- [ ] 11. Checkpoint — API server
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 12. Web UI serving and chat mode
  - [ ] 12.1 Add static file serving to the API server
    - `GET /*` serves files from the bundled `web/` asset directory (embedded in binary via `include_dir!` or equivalent)
    - If no web assets are bundled: return 404 for all non-API routes
    - _Requirements: 7.1, 7.7_
  - [ ] 12.2 Implement chat mode startup in `runtime/main.rs`
    - After the API server is listening: if mode == chat and web assets bundled, open browser via `cmd /c start http://localhost:<port>`
    - If mode == chat but no web assets: log WARNING "No UI assets bundled, running in server-only mode"
    - _Requirements: 7.1, 7.7_
  - [ ] 12.3 Build the Web UI markup and layout
    - Chat input field and submit button
    - Conversation history panel
    - Configuration summary bar (fetched from `/health`)
    - Tokens/sec display placeholder
    - Stop button
    - _Requirements: 7.2_
  - [ ] 12.4 Implement Web UI streaming and interaction logic
    - Submit sends `POST /v1/chat/completions` with `stream: true` via Fetch API
    - Parse SSE `ReadableStream`; append tokens to history panel as they arrive
    - Update tokens/sec display from streaming stats
    - Stop button calls `AbortController.abort()`
    - Maintain in-memory message history for the session
    - Bundle `marked.js` inline for markdown rendering
    - _Requirements: 7.3, 7.4, 7.5, 7.6_

- [ ] 13. CLI flags and mode selection
  - Add to `runtime` binary via `clap`: `--chat`, `--server`, `--port <n>`, `--host <addr>`, `--gpu-layers <n>`, `--context <n>`, `--backend cpu|cuda`
  - Mode flags override the packaged manifest default
  - Hardware override flags (`--gpu-layers`, `--context`, `--backend`) apply after the Configurator's automatic calculation
  - _Requirements: 4.4, 8.1, 8.2, 8.3, 8.4, 8.5_

- [ ] 14. Structured logger, graceful shutdown, and error display
  - [ ] 14.1 Implement structured logger in `core`
    - Write timestamped log lines to `<exe_dir>/runtime.log`; fall back to `%TEMP%/localai_<name>/runtime.log` if not writable
    - Log format: `[YYYY-MM-DD HH:MM:SS] [LEVEL] <message>`
    - On unrecoverable error: print log file path and brief message to stderr before exit
    - _Requirements: 10.3, 10.4_
  - [ ] 14.2 Implement graceful shutdown in `runtime/main.rs`
    - Register Ctrl+C handler via `ctrlc` crate
    - Stop accepting new requests; wait up to 30s for in-flight inference; cancel queued requests; free llama model and session; exit 0
    - _Requirements: 9.1, 9.2, 9.3, 9.4_

- [ ] 15. Checkpoint — runtime executable
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 16. Packager CLI
  - [ ] 16.1 Implement `packager` binary
    - Accept all flags via `clap`: `--model`, `--name`, `--mode`, `--context`, `--ui-assets`, `--api-key`, `--output`
    - Validate GGUF via `core` validator; descriptive error to stderr + non-zero exit on failure
    - If `--mode chat` and `--ui-assets` not provided: error and exit
    - Hash API key as `SHA-256(random_16_byte_salt || key)`; store as `"sha256:<hex_salt>:<hex_digest>"`
    - Embed GGUF, DLLs (from local CUDA 12 SDK), web assets, and `manifest.json` as Win32 resource sections in the output `.exe`
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6_
  - [ ]* 16.2 Write unit tests for Packager
    - Valid GGUF + all required args → output bundle contains correct manifest fields
    - Invalid GGUF (wrong magic) → error to stderr, non-zero exit, no output file
    - `--mode chat` without `--ui-assets` → error to stderr, non-zero exit
    - _Requirements: 5.4, 5.5_

- [ ] 17. Final checkpoint — end-to-end
  - Ensure all tests pass, ask the user if questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- Property tests reference numbered properties from `design.md` and run >= 100 iterations via `proptest`
- Tag format: `// Feature: local-ai-runtime, Property N: <title>`
- The chosen llama.cpp crate is `llama-cpp-rs`; do not use `llama-sys`
- The Self-Extractor (Task 2) is intentionally scaffolded before all other Runtime components
