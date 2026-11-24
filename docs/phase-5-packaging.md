# Phase 5 — Packaging and publish readiness

Goal: a Windows installer a stranger can install and use safely.

## Deliverables
1. **Installer** with `electron-builder` (NSIS): `oneClick: false`, `allowToChangeInstallationDirectory: true`, per-user by default, desktop and start-menu shortcuts, Arabic + English installer (`multiLanguageInstaller`). Script `npm run dist` outputs `dist/The-Knower-Setup-<version>.exe`. The installer must include `node-llama-cpp` and its prebuilt binaries (external dependency + `asarUnpack`, see Phase 3). Verify FIRST, with an unpacked build, that the packaged app loads a model, before polishing anything else.
2. **Icons**: script that generates `.ico`/`.png` from `resources/logo.svg` (placeholder logo stays replaceable by swapping that one file).
3. **First-run onboarding** (3 steps, shown once): language → terms (must scroll to the end to enable "Accept") → environment check (the Doctor, with an "Install REA" button).
4. **Terms and disclaimer** in `resources/terms.en.md` and `resources/terms.ar.md`, shown in onboarding and in Settings → About & terms. Content: lawful use only (authorized software, interoperability, education, preservation); user is responsible for how results are used; no warranty; the app does not bundle, crack, or bypass any third-party tool or license; third-party analysis tools (e.g., Hopper, IDA) are not included and must be obtained under their own licenses; no output is legal advice. Keep it plain and short (under 1 page per language). Add a clearly marked line: "Have a lawyer review before commercial release."
5. **Credits**: About page shows version, "Uses REA (Reverse Engineer Anything) by morluto, MIT license", and "The Knower is independent and not affiliated with REA". Generate `THIRD_PARTY_NOTICES.md` from the installed dependencies' license files with a script (`npm run notices`), and include REA's MIT license text. Ship both inside the installer.
6. **Logs and data**: local-only log file in `userData/logs` (rotate at 5 MB, no keys, no project content). Settings → Data: show where data lives, "Open logs folder", "Delete app data" (confirm dialog; does not touch project folders).
7. **Repo hygiene**: `README.md` (what it is, install, first analysis walkthrough, privacy: no telemetry, everything including the AI model runs on the user's machine), `LICENSE` (leave the choice as a TODO for the owner), `.gitignore`, screenshot placeholders.
8. No auto-update in the MVP.

## Known limitation to document in README
The installer is unsigned, so Windows SmartScreen shows a warning on first run. Code signing is deferred.

## QA checklist (test on a clean Windows VM or fresh user profile)
1. Install to a non-default folder with a space in the path.
2. First run: onboarding appears, Arabic is default, RTL correct.
3. Node.js missing → Doctor explains and links to nodejs.org; after installing Node, re-check passes.
4. Install REA from the app, analyze an **open-source** Electron app, run interpret → spec → lint → export.
5. Build session cannot see evidence (spot-check the request body).
6. Uninstall removes the app; app data stays unless the user deleted it in Settings.
7. Disconnect the network: UI, fonts, and local-model chat still work.
8. Test on a machine with 8 GB RAM using a small model (about 2B parameters, Q4): loading, chat, and one interpretation unit work without crashing.

## Acceptance criteria
- [ ] Installer builds and installs; chosen install directory is respected.
- [ ] Onboarding gates on terms acceptance and runs only once.
- [ ] Terms, credits, THIRD_PARTY_NOTICES present in the installed app.
- [ ] Logs contain no keys or project content (grep a full session's logs).
- [ ] Full QA checklist above passes; list any failures in the report.
