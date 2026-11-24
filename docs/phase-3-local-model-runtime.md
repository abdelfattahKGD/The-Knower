# Phase 3 — Local model runtime and chat

Goal: run GGUF models **directly inside the app** with the built-in llama.cpp engine (no other software required), keep the engine layer pluggable for future updates, stream chat, manage models, persist sessions.

## Design decisions (do not change without asking)
1. **One engine now: `llamacpp`** (built in, GGUF). Define a small `ChatEngine` interface and registry so more engines can be added in future updates (after the MVP is tested and released), but implement ONLY `llamacpp`. Do NOT implement an OpenAI-compatible/server engine, ONNX, MLX, or anything else, and write no provider, API-key, or network-model code in the MVP. Details in section 1b.
2. **Library: `node-llama-cpp`** (llama.cpp bindings with prebuilt binaries; also what Electron's own experimental `@electron/llm` uses). Pin an exact version that was published at least 14 days ago and report it.
3. **Inference runs in an Electron `utilityProcess`** (not main, not renderer), so a crash or heavy load never freezes the UI. Main talks to it by messages and forwards events to the renderer.
4. **Packaging is the known risk.** `node-llama-cpp` is ESM-only and has native binaries: load it with dynamic `import()` in the utility-process entry, mark it **external** in the electron-vite config (do not bundle it), and unpack it from asar (`asarUnpack`). Verify details against the docs of the installed version. **Do an `electron-builder --dir` smoke test that loads a model BEFORE building the rest of this phase.** If you cannot make the packaged build load a model after 3 attempts, STOP and report the exact error. Do NOT silently switch approach (the owner's fallback would be bundling the `llama-server` binary as a hidden sidecar; that is the owner's decision).
5. **No tool/function calling, ever.** The app drives the workflow; the model gets text prompts and returns text. (Small models are unreliable at tool calling.)

## Deliverables

### 1) Runtime (`src/main/runtime/` + utility process entry)
API exposed to the rest of main: `loadModel(path, opts)`, `unloadModel()`, `chat({ systemPrompt?, messages }, { maxTokens, temperature, signal })` → async stream of text deltas, `countTokens(text)`, `status()`. One model loaded at a time; unload on quit; kill the utility process on quit.
Hints (names must be verified against the installed version's docs): `getLlama()`, `llama.loadModel({ modelPath, gpuLayers })`, `model.createContext({ contextSize })`, `LlamaChatSession` with a context sequence, `session.prompt(text, { onTextChunk, signal, maxTokens, temperature })`.
- Each independent task (e.g., interpreting one unit in Phase 4) uses a **fresh session/sequence** so history never accumulates. UI chat sessions replay their saved history explicitly.
- Backend: let the library choose the best available (CPU/GPU) automatically. If a GPU backend fails to load, fall back to CPU and tell the user in the UI.

### 1b) Engine interface and registry
```ts
interface ChatEngine {
  id: string;                       // "llamacpp"
  label: string;
  load(entry: ModelEntry, s: RuntimeSettings): Promise<void>;
  unload(): Promise<void>;
  chat(req: ChatRequest, opts: { maxTokens: number; temperature: number; signal: AbortSignal }): AsyncIterable<string>;
  countTokens(text: string): Promise<number>;
}
```
- The rest of the app (chat UI, Phase 4 interpretation) talks ONLY to the registry's active engine; no llama-specific code outside `engines/` and the utility-process entry.
- Unknown file type chosen in "Local file": reject with this message (translated): "This version supports GGUF models. Support for other formats is planned in future updates." Never a vague error.

### 2) Model manager (`userData/models.json`)
- Entry: `{ id, name, engine: "llamacpp", path, sizeBytes, source: "import"|"download", addedAt }`.
- **Import:** pick a `.gguf` file; by default keep it where it is (reference its path), with an option "Copy into app folder" (progress bar).
- **Download from URL:** direct file URL → stream to a temp file → rename; progress, cancel, resume with `Range` when the server supports it, free-disk-space check, optional SHA-256 field verified after download.
- Validate every file: first 4 bytes must be `GGUF`; otherwise reject with a clear message.
- `resources/curated-models.json`: an empty array with the schema `{ name, sizeBytes, url, sha256, notes }` and a TODO comment for the owner to fill. **Do NOT invent model URLs.**
- Removing an entry deletes the file only if the app downloaded or copied it (confirm dialog).

### 3) Runtime settings (`userData/runtime.json`)
`contextSize` (default 4096), `maxOutputTokens` (default 1024), `temperature` (default 0.2, analysis needs low randomness), `gpuLayers` (Auto), `threads` (Auto). Before loading, estimate memory (file size + rough KV cache) and warn if it exceeds ~70% of free RAM, suggesting a smaller model or a lower context size.

### 4) Token budgeting (`src/main/services/tokens.ts`)
- `countTokens`: use the loaded model's tokenizer; fallback `ceil(chars / 3.2)` when no model is loaded.
- `chunkText(text, maxTokens, overlapTokens = 0)`: split on paragraph/line boundaries first; split mid-line only if a single line exceeds the limit.
- `inputBudget(settings)` = 60% of `contextSize` minus `maxOutputTokens`, never below 500.

### 5) Real UI
- Settings → Models (built as mock in Phase 1) becomes real: add/import/download, progress, remove, active model, runtime settings form.
- Composer model pill lists installed models; selecting one loads it (progress, "Loading model…" state, memory warning).
- Chat end to end: send → streamed answer → Stop button → friendly errors with fix hints: file missing, corrupt/not-GGUF, out of memory (suggest smaller model/lower context), backend failure (CPU fallback notice), model not loaded.
- First-run empty state in Models: "No model yet" with the two ways to add one.

### 6) Project and sessions
"Open folder" sets the current project and creates `.knower/project.json` if missing. Sessions are saved to `.knower/sessions/<id>.json` after each completed assistant message (atomic write: temp file + rename). Real session list in the sidebar with rename and delete. If no project is open when the user sends a message, ask them to open one first.

## Tests (Vitest, no real model needed)
GGUF magic-byte validation; `chunkText`; `inputBudget`; download resume logic against a mocked HTTP server; atomic session write; utility-process message protocol using a fake engine.

## Acceptance criteria
- [ ] On a machine WITHOUT LM Studio/Ollama: import a small GGUF (about 2B parameters, Q4) and chat; text streams token by token.
- [ ] **Packaged build** (`electron-builder --dir`) launches and loads a model (report how you configured external/asarUnpack).
- [ ] Stop cancels generation in about a second.
- [ ] Killing the utility process shows an error and the model can be reloaded without restarting the app.
- [ ] Loading a model too large for the machine shows the memory warning / friendly out-of-memory error instead of crashing the app.
- [ ] Works with the network disconnected; verify that NO network request is made during chat or model loading.
- [ ] Download supports cancel and resume; a non-GGUF file is rejected with the message described in 1b.
- [ ] Adding a hypothetical third engine requires only a new file in `engines/` plus a registry entry (state in the report which files you touched to prove it with a dummy engine in a test).
- [ ] Sessions persist across restarts; writes are atomic.
- [ ] No tool-calling code exists (grep); tests pass.
