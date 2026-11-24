# Phase 1 — Shell and UI (no backend)

Goal: a polished, fully navigable desktop UI with mock data. No REA, no model calls yet.

## Deliverables
1. Project scaffold (electron-vite + React + TS), scripts: `dev`, `build`, `typecheck`, `test`.
2. Secure BrowserWindow (rules in CLAUDE.md), default 1280×800, min 960×600, native frame, application menu removed, title "العارف — The Knower".
3. Design system (tokens + base components).
4. App layout: sidebar, Home, Session view, right panel, Settings pages — all with mock data in `src/renderer/mock/`.
5. i18n (ar default, en) with live switch and RTL/LTR flip. Language saved in `localStorage`.
6. `src/shared/brand.ts` exporting `{ nameEn: "The Knower", nameAr: "العارف" }`; `resources/logo.svg` is a simple purple placeholder mark (an eye inside a rounded square). Everything references these two only.

## Design tokens (`tokens.css`) — dark theme only in the MVP, structured so a light theme can be added later
```css
:root{
  --bg:#15131b; --bg-sidebar:#1b1822; --surface:#201d29; --surface-2:#292535; --surface-hover:#2f2a3d;
  --border:#34303f; --border-strong:#463f57;
  --text:#ece9f3; --text-muted:#a29bb5; --text-faint:#756e89;
  --accent:#9277f2; --accent-hover:#a38cf5; --accent-strong:#7a5ce6; --accent-soft:rgba(146,119,242,.14);
  --success:#4cc38a; --warning:#e5b13a; --danger:#ef6b6b; --info:#5aa9f0;
  --radius-sm:8px; --radius-md:12px; --radius-lg:18px; --radius-pill:999px;
  --shadow-pop:0 10px 30px rgba(0,0,0,.45);
  --font-ui:"IBM Plex Sans Arabic","Segoe UI",Tahoma,system-ui,sans-serif;
  --font-mono:"Cascadia Code",Consolas,"Courier New",monospace;
  --dur:150ms; --ease:cubic-bezier(.2,.7,.2,1);
}
```
Type scale: base 15px/1.65; small 13px; title 20px/600; page greeting 28px/600. Primary buttons use `--accent-strong` background with white text (contrast). Focus ring: 2px `--accent` outline with 2px offset on every focusable element. Respect `prefers-reduced-motion`. Motion only for user actions (open/close panel, expand card) — 150ms; no entrance animations on page load.

## Layout
```
┌ sidebar 280px ┬────────── main ──────────┬ right panel 360px (collapsible) ┐
```
In RTL the sidebar sits on the right and the panel on the left (use flex order + logical properties, do not duplicate CSS).

### Sidebar (collapsible with button and `Ctrl+B`)
- Top: logo + app name.
- Primary button "New investigation" (accent-strong).
- Project block: folder icon + folder name chip + "Open folder" text button.
- Session list grouped "Today" / "Earlier"; row = title (1 line, ellipsis) + small stage dot; hover shows `--surface-hover`; selected row has `--accent-soft` background and a 3px inline-start accent bar.
- Bottom: Settings button, language toggle (AR | EN).

### Home page
Centered column, max-width 720px, vertically centered. Greeting (ar: "ما الذي تريد أن نفكّكه اليوم؟", en: "What shall we take apart today?"). Below it the **composer card**: radius 18px, 1px border, `--surface` background, auto-growing textarea (max 8 lines), bottom row = attach button (+) · model selector pill (mock text "gemma-4-e2b · Local") · send button (circle, accent-strong, disabled when empty). Under the composer: 3 suggestion chips that fill the textarea: "Analyze an Electron app", "Explain one module", "Write the clean-room spec" (translated).

### Session view
- Top bar: session title (editable on click) + **stage stepper**: Authorize → Analyze → Interpret → Spec → Build. Pill-shaped steps; done = check icon + muted; current = `--accent-soft` fill + accent text; upcoming = faint. This stepper is the one distinctive element of the app: keep everything else quiet.
- Message column centered, max-width 760px. User message: right-aligned in LTR (inline-end), `--surface` bubble, radius 14px. Assistant message: no bubble, markdown rendered (headings, lists, tables, code with highlight.js, inline code in mono with `--surface-2` background).
- **Tool card** (for REA/command steps): collapsible card; header = icon + title ("Analyze JavaScript application") + status (spinner / check / error) + duration; body = mono output block, max-height 240px, scroll. Collapsed by default when finished.
- **Evidence chip**: small pill `file:line` style (e.g. `main.js · module 12`) inline in assistant text; click opens the right panel at that item.
- Composer fixed at the bottom (same component as Home).
- Right panel with tabs: "Evidence" (list of modules with search box) and "Spec" (markdown preview of mock spec). Collapsible; default closed.

### Settings page (left nav + content)
Sections: Models, Environment, Language & appearance, About & terms. Build the **forms and layout** with mock values; no real behavior. Models: list of installed model cards (name, file size, quantization, "Active" badge, Remove) + "Add model" button (dialog with two tabs: "Local file (.gguf)" and "Download from URL") + a "Runtime" form (context size, max output tokens, temperature, GPU layers: Auto/number, threads: Auto/number). Environment: list of check rows (label, status badge, details, "Re-check" button) with 4 mock rows (Node.js, npm, REA, REA doctor).

## Base components to build (each with default/hover/focus/disabled states)
Button (primary/secondary/ghost/danger), IconButton, Input, Textarea, Select, Toggle, Badge, Chip, Card, Modal (focus trap, Esc closes), Tooltip, Tabs, Stepper, Spinner, Toast, EmptyState (message + one clear action).

## Copy rules
Sentence case. Buttons say what happens ("Open folder", "Run analysis"). Errors say what went wrong and how to fix it. Empty states invite an action. Provide every string in both ar.json and en.json.

## Acceptance criteria
- [ ] `npm run dev` opens the window; no console errors; security settings per CLAUDE.md verified in code.
- [ ] Switching AR/EN flips direction correctly: sidebar side, message alignment, chevrons, send arrow, paddings.
- [ ] Home, Session, Settings (4 sections), right panel, sidebar collapse (`Ctrl+B`) all work with mock data.
- [ ] Stepper shows 5 stages with the 3 visual states.
- [ ] Tool card collapses/expands; evidence chip opens the right panel.
- [ ] All base components exist and show a visible focus ring via keyboard.
- [ ] No hard-coded user-facing strings outside the two JSON files; no `left`/`right` in CSS.
- [ ] Fonts load offline (disconnect network and restart to verify).
