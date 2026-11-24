# Phase 4 — Clean-room workflow

Goal: authorize → analyze → interpret → write a behavioral spec → hand off a clean package, with the separation between analysis and build enforced by the app.

## Stage 0 — Authorization gate
Modal shown before the first analysis of a target. Fields: target path (read-only), basis (select: "I own it" / "I have written permission" / "Open-source or its license allows it" / "Research or interoperability under applicable law" / "Other"), free-text scope note, and a required checkbox: "I am authorized to analyze this software and I take responsibility for how I use the results." Saved to `.knower/authorization.json` with timestamp and app version. The Analyze action stays disabled until it exists. Append a line to `.knower/process-log.jsonl` for every stage start/finish (stage, time, counts; no content).

## Stage 1 — Analyze
Phase 2 functionality, now behind the gate.

## Stage 2 — Interpret (per unit, map step)
- Build the unit list from `EvidenceIndex`, ordered: main-process entry and IPC handlers first, then preload, then renderer modules, then libraries (vendor/node_modules-like units last and skipped by default with a toggle).
- For each unit: get its evidence text, split with `chunkText` to fit `inputBudget`, call the model sequentially (concurrency 1), and require this exact markdown output:
```
## Purpose
## Inputs
## Outputs
## Side effects
## Dependencies
## Evidence refs
## Unknowns
```
- System prompt (put it in `src/main/prompts/interpret.ts`): "You describe what a software unit DOES, using only the provided evidence. Never guess. If something is not in the evidence write it under Unknowns. Describe behavior and interfaces in plain language. Do not reproduce code." Keep the prompt short and in English; the user-facing output language follows the chosen UI language only in the final spec.
- Post-validation in code (not by the model): every ref listed under "Evidence refs" must exist in the evidence index; unknown refs get an "unverified" badge. Missing sections → one automatic retry, then mark the unit "needs review".
- Save to `.knower/interpretations/<unit-id>.md`. UI: a progress list (pending / running / done / needs review), Pause, Cancel, and Resume that skips finished units.

## Stage 3 — Spec (reduce step)
- Map-reduce the interpretations into `.knower/spec/spec.md` with these sections: 1 Overview and purpose · 2 Architecture (processes, windows) · 3 Features (user-visible behavior) · 4 Data and storage formats · 5 Interfaces (IPC messages, network endpoints, file formats — described as behavior) · 6 Non-functional notes · 7 Open questions.
- Generation rules (system prompt in `src/main/prompts/spec.ts`): describe behavior and interfaces, not implementation; no code; no copied identifiers except public names needed for interoperability (endpoint paths, protocol/format names); no long verbatim strings.
- **Spec lint** (deterministic code, `src/main/services/spec-lint.ts`) producing a report: (a) any code fence in the spec; (b) any run of ≥ 12 consecutive identical tokens shared between the spec and the evidence/interpretations text; (c) string literals longer than 80 chars. Each finding has location + excerpt. The user must fix or explicitly accept each finding (with a one-line note stored in the process log) before Export is enabled.
- The spec is shown and editable in the right panel "Spec" tab (plain markdown textarea + preview toggle), saved on change.

## Stage 4 — Clean build
1. **Export clean-room package** to `.knower/handoff/` (or a folder the user picks): `spec.md`, `README.md` (how to use the spec in any coding tool), `CLEANROOM-NOTICE.md` (process record: authorization summary, stage dates, lint result, statement that the package contains no code or evidence from the target). The package contains NO evidence, interpretations, or raw REA output.
2. **Clean build session**: a session of type `build`. Its context is assembled by ONE function `assembleBuildContext(spec, chatMessages)` that can only read `spec.md` and the chat. It never touches evidence/interpretation folders or the target path. Unit test: assert that content from the evidence folder never appears in the assembled context, and that the function has no file-system access to those folders. UI marks the session with a "Clean room" badge and a distinct accent border.
3. Code blocks in a build session have a "Save to file" button that writes only inside `<project>/build/` (sanitize the filename; reject `..` and absolute paths).

## Tests (Vitest)
Lint rules with fixtures (positive and negative), ref validation, `assembleBuildContext` isolation, authorization gate blocks analysis, path sanitization for "Save to file", resume logic.

## Acceptance criteria
- [ ] Analysis cannot start without a saved authorization file.
- [ ] Interpretation runs on a real open-source Electron app with a small local model; units finish, bad refs are flagged "unverified".
- [ ] Spec is generated; lint catches a deliberately planted code fence and a 12-token copy.
- [ ] Export is blocked until findings are resolved/accepted, then produces the 3-file package with no evidence inside (verify by search).
- [ ] Build session never includes evidence (test + manual inspection of the actual request body).
- [ ] "Save to file" cannot write outside `build/`.
- [ ] `process-log.jsonl` contains the stage events.
