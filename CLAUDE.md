# The Knower (العارف) — Project Rules

Read this file fully before any task. Then read ONLY the phase spec you were given in `docs/`.

## What we are building
A Windows-first desktop app (Electron) that helps a person reverse engineer **Electron / JavaScript desktop apps** with AI, then write a **clean-room behavioral spec** so a new app can be built without copying the original.
- Models: **local only, run directly inside the app** with the built-in llama.cpp engine (`node-llama-cpp`, GGUF files, in an Electron utility process). No LM Studio, no Ollama, no external server, no cloud APIs, no API keys in the MVP. The engine layer is a small pluggable interface (`ChatEngine`) so more engines can be added in future updates, but ONLY the llama.cpp engine is implemented now. The app is only the harness; it is NOT a model.
- Reverse engineering engine: the open-source **REA** CLI (npm package `rea-agents`, repo github.com/morluto/rea, MIT). We call its CLI. We do NOT use MCP in this project.
- UI: same structure and feel as Claude Code desktop, but the accent color is **purple** (never orange). Arabic RTL-first, English supported.
- Distribution: public .exe installer (MVP). Name: "The Knower" / "العارف". Logo is a placeholder, replaceable by swapping one file.

## Hard rules (follow without exception)
1. Implement ONE phase at a time. Never build features of later phases. Never add features not in the spec. If something is unclear, choose the simplest option and list it under "Assumptions" in your report.
2. NEVER invent REA CLI commands, flags, or JSON fields. Run `rea --help` and `rea <command> --help` and read the real output. If the real output differs from the spec, trust the real output and report the difference.
3. No telemetry, no analytics, no CDN/remote assets. The app must work offline except for the one-time REA install and optional model downloads.
4. Electron security (mandatory): `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true` for renderer, preload exposes a minimal typed API through `contextBridge`, every IPC input is validated in the main process, strict CSP, never `shell: true`, always `spawn/execFile` with an args array, validate and normalize every path, NEVER execute or open the analyzed target.
5. Privacy: nothing leaves the machine. No network call may ever carry user project content or analysis output. The only allowed network use is the one-time REA install and optional model downloads (which carry no project data).
6. Windows paths: use the `path` module; test paths with spaces and Arabic characters.
7. i18n: no hard-coded UI strings. Use `src/renderer/i18n/ar.json` and `en.json`. Layout uses CSS logical properties (`margin-inline-start`, `padding-inline-end`, `inset-inline-start`, `text-align: start`), never `left/right`. Icons that imply direction (chevrons, send arrow) flip in RTL.
8. Few dependencies. No native Node modules (no better-sqlite3 etc.) — they break packaging. The ONE exception is `node-llama-cpp` (prebuilt binaries), handled exactly as described in `docs/phase-3-local-model-runtime.md`.
9. TypeScript `strict: true`. Small files. No component over ~250 lines.
10. Product stance: the app is for lawful analysis (authorized software, interoperability, education, preservation). Do NOT implement anything for bypassing licenses/DRM, cracking, patching binaries, malware, or exploit generation.

## Tech stack
- Electron + `electron-vite`, React 18, TypeScript, plain CSS with design tokens (`src/renderer/styles/tokens.css`), Zustand (state), `lucide-react` (icons), `react-markdown` + `remark-gfm`, `highlight.js` (core + few languages), Vitest (main-process unit tests), `node-llama-cpp` (local inference, Phase 3), `electron-builder` (Phase 5).
- Font: bundle `@fontsource/ibm-plex-sans-arabic` (weights 400/500/600) so it works offline. Mono: Cascadia Code → Consolas fallback.

## Repository layout
```
the-knower/
  CLAUDE.md
  docs/                      (phase specs)
  resources/                 (logo.svg, icons, terms.*.md)
  src/shared/                (brand.ts, types.ts, ipc-channels.ts, rea-version.ts)
  src/main/                  (index.ts, ipc/, services/)
  src/preload/index.ts
  src/renderer/              (App.tsx, components/, pages/, store/, i18n/, styles/, mock/)
```
Per-project data lives in the folder the user opens, under `.knower/`:
```
.knower/project.json  authorization.json  process-log.jsonl
.knower/sessions/*.json  evidence/*.json  interpretations/*.md  spec/spec.md  handoff/
```
App-wide settings live in `app.getPath('userData')`.

## Definition of done (every phase)
- `npm run typecheck` passes, `npm run dev` launches without console errors.
- Every acceptance criterion in the phase spec is checked (manually or by test).
- Final report in exactly this format:
  1. Files created/changed
  2. How to run
  3. Acceptance checklist (each item PASS / FAIL + one line why)
  4. Assumptions you made
  5. Known issues
- If you are stuck on an error for more than 3 attempts, STOP and report the exact error text instead of trying random fixes.

## Clean-room principle (product-critical)
Three separated stages: **Analysis** (sees the target) → **Spec** (describes behavior only, no code) → **Build** (sees ONLY the spec). The app enforces the separation technically (see Phase 4). Never leak evidence or target code into the Build stage.
