# Phase 2 — REA bridge and environment doctor

Goal: the app installs and runs REA locally, shows a real environment check, and analyzes an Electron/JavaScript app, saving the evidence.
Scope: **Electron / JavaScript targets only.** Native, .NET, APK, firmware are out of scope.

## Facts you may rely on (from REA's README)
- REA is the npm package `rea-agents`. Needs Node.js `^22.19.0 || ^24.11.0 || >=26` and npm.
- A documented JS/Electron command: `npx -y rea-agents@latest analyze-javascript-application /absolute/path/to/app --json` (target = extracted app directory or an `.asar`).
- `rea doctor --json` exists. Analysis runs locally; results include evidence and limitations.
- Everything else (other commands, flags, JSON shape) you must discover from `rea --help` and from real output. Do not guess.

## Deliverables

### 1) Version pinning
`src/shared/rea-version.ts` exports `REA_VERSION`. Run `npm view rea-agents time --json`, choose the **newest version published at least 14 days ago**, and put it there. Report the chosen version and its publish date.

### 2) `ReaService` (main process, `src/main/services/rea.ts`)
- Install dir: `app.getPath('userData')/rea/` with its own `package.json`. Install with `npm install rea-agents@<REA_VERSION> --save-exact` run through `spawn` (args array) using the system npm. Stream install logs to the UI.
- Running the CLI: resolve the entry file from `node_modules/rea-agents/package.json` → `bin` field and run it as `node <entry> <args>` with `spawn` and **no shell** (spawning `.cmd` files without a shell fails on modern Node on Windows). Use the system `node`.
- `getStatus()`, `install()`, `update()` (reinstalls the pinned version only — no "latest"), `runDoctor()`, `analyzeJavaScriptApplication(targetPath, opts)`.
- Process control: configurable timeout (default 10 min), cancel button (kill the whole process tree: `taskkill /PID <pid> /T /F` on Windows), stdout/stderr streamed to a file on disk (analysis JSON can be many MB — never hold it all in memory or send it whole over IPC).
- Output saved to `<project>/.knower/evidence/analysis-<ISO timestamp>.json` plus a `.log` file with stderr.

### 3) Environment doctor
Checks, each returns `{ id, status: ok|warn|error, title, detail, fixHint }`:
1. Node.js present and version satisfies the range above (fixHint: link to nodejs.org).
2. npm present.
3. REA installed and installed version equals `REA_VERSION`.
4. `rea doctor --json` result (parse defensively; show raw text if parsing fails).
The Settings → Environment page (built mock in Phase 1) now shows real results, a "Re-check" button, and "Install REA" / "Reinstall REA" with a live log box.

### 4) Target handling
`findElectronApp(dir)` returns the analyzable path: checks `<dir>/resources/app.asar`, `<dir>/resources/app/`, `<dir>/app.asar`, or `<dir>` itself if it contains `package.json`. If nothing matches, return a clear error (what was searched). The UI lets the user pick a folder or an `.asar`. The target is only ever read; never modify, copy into, or execute it.

### 5) Evidence adapter — discovery first
Run the analysis on a small **open-source** Electron app. Write what you actually observed to `docs/rea-output-notes.md` (top-level keys, how modules/imports/IPC boundaries are represented, size, runtime). Only then write `src/main/services/evidence.ts`, which **defensively** parses the output into:
```ts
type EvidenceIndex = { units: { id: string; title: string; kind: string; sizeBytes?: number; refs: string[] }[]; raw: unknown; limitations: string[] };
```
Unknown fields are preserved in `raw`. If a field you expected does not exist, say so in the report — do not fabricate it.

### 6) UI wiring
In the Session view, the "Analyze" action: pick target → run → a tool card shows live status/log → on success a summary (counts of units, limitations) and the Evidence tab in the right panel lists the units with a search box. (The authorization gate comes in Phase 4; for now a plain confirm dialog is enough.)

### 7) IPC (typed, in `src/shared/ipc-channels.ts`)
`rea:status`, `rea:install`, `rea:doctor`, `rea:analyze`, `rea:cancel`, `rea:log` (event), `project:open`, `target:find`. Every payload is validated in main.

## Tests (Vitest)
Node version range check; `findElectronApp` (4 layouts + none found); path validation (reject relative paths, UNC oddities, non-existing paths); evidence adapter with a tiny fixture.

## Acceptance criteria
- [ ] Doctor page shows real results for all 4 checks with correct statuses (test once with Node missing from PATH if possible, or mock the check).
- [ ] "Install REA" installs the pinned version into `userData/rea` and the status flips to ok.
- [ ] Analysis of an open-source Electron app completes, evidence JSON is saved under `.knower/evidence/`, the UI shows the unit list.
- [ ] Cancel really kills the process tree (verify in Task Manager).
- [ ] Paths with spaces and Arabic characters work.
- [ ] A 50 MB+ output does not freeze the UI.
- [ ] `docs/rea-output-notes.md` exists and matches what you observed.
- [ ] Tests pass; no `shell: true` anywhere (grep).
