# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Daily Tracker — a single-file, installable PWA (no backend, no build step) hosting multiple small self-contained "tools" behind one shared shell. Currently: a Calorie/weight-loss Calculator and a Press-Ups tracker. Ships as static files served directly over HTTPS (GitHub Pages, Netlify, etc.) and installs to a phone home screen via "Add to Home Screen" or a PWABuilder-generated APK.

## Files

- `index.html` — the entire app: markup, CSS, and JS in one file (~1700 lines). Virtually all work happens here.
- `manifest.json` — PWA manifest (name, icons, display mode).
- `sw.js` — service worker, cache-first with network fallback, for offline installability.
- `icon-192.png` / `icon-512.png` — app icons. There's no committed generator script; they were produced ad hoc with a short Pillow (PIL) script when last rebranded — regenerate similarly if needed.
- `README.md` — end-user install guide (Add to Home Screen / PWABuilder→APK) and the Cloud sync (Firebase) setup walkthrough, including the exact Firestore security rules in use. Read this before changing hosting, PWA, or Firebase-adjacent behavior.

## Commands (no build system)

There's no `package.json`, bundler, linter, or test suite — it's hand-written vanilla HTML/CSS/JS. The workflow used throughout this project instead:

- **Syntax-check the inline script** after any edit: extract the contents of the single `<script>...</script>` block that has no `src` attribute (the three Firebase `<script src=...>` tags before it are plain includes and load nothing inline), and run `node --check` on the extracted text.
- **Serve locally** to confirm the static files resolve correctly: `python3 -m http.server` from the project folder, then request `index.html`, `manifest.json`, `sw.js`, and both icons and confirm 200s.
- **No automated test harness.** Calculation changes are validated with small ad hoc Node scripts that replicate the relevant pure functions (BMR/TDEE math, regression, date-grouping, target-date math, etc.) and assert against hand-computed expected values, run with `TZ=Europe/London node ...` when the logic touches dates/timezones. This is how every formula in this codebase was checked during development — follow the same approach for new math rather than trusting it by inspection.

## Architecture

### Single-file structure

Everything lives in `index.html`: one `<style>` block, all markup in `<body>`, all JS in one `<script>` block at the bottom. There is no module system — every function and top-level `const`/`let` is a global, so name collisions matter when adding new tools.

### Multi-tool shell

The app hosts multiple independent "tools" behind one shared bottom nav (Today / History / Settings) and header. Which tool is showing is controlled by:

- `TOOLS` — an array of `{key, label, icon}`, driving the hamburger-menu list.
- `activeTool` — a plain variable, persisted to its own localStorage key (`dailyTrackerActiveTool_v1`), deliberately **not** part of the synced `DATA` blob. Switching tools should never trigger a cloud write, and signing in on another device shouldn't change which tool you land on.
- Every view section (`#view-today`, `#view-history`, `#view-settings`) contains one `<div class="tool-panel" data-tool="...">` per tool. `applyToolVisibility()` toggles which panel is visible by matching `data-tool` against `activeTool`. The bottom-nav / `goTo(view)` logic is unaffected by which tool is active — it only ever toggles between today/history/settings.
- `render()` unconditionally re-renders every tool's content into its own panel each time (cheap given the data size), then calls `applyToolVisibility()` — switching tools is purely a visibility flip, not a re-render.
- Settings' "Cloud sync" and "Data" (export/import/erase) cards are **shared**, sitting outside any `tool-panel` — they operate on the whole unified `DATA` object, not a single tool.

### Adding a new tool

Since more tools are planned, the established pattern (used when Press-Ups was added alongside the Calorie Calculator) is:

1. Add `{key, label, icon}` to `TOOLS`.
2. Add a `<div class="tool-panel" data-tool="yourkey">` inside each of the three `<section class="view">` blocks.
3. Give the tool its own namespaced slice of `DATA` (e.g. `DATA.yourTool = {settings:{...}, ...}`), with a `defaultYourTool()` factory and a migration fallback wired into all four places existing data can arrive from: `loadData()`, the remote-adopt branch inside `syncOnSignIn()`, `importData()`, and `clearAllData()`. The Press-Ups additions in each of those four spots show the exact pattern to copy.
4. Write `renderYourToolToday/History/Settings()` functions and call them from `render()`, plus from `goTo('history')` if the tool has a History chart.
5. Reuse `RANGE_OPTIONS` + a `windowedDaily`-style helper (see `pressUpsWindowedDaily`) for any date-bucketed history/chart, rather than inventing new range logic.

### Data & persistence

- A single `DATA` object holds everything (`settings` + `entries` for the calorie tool, `pressUps` for the press-up tool), serialized as one blob to `localStorage` under `STORAGE_KEY = 'quickCalorieData_v1'` (the key name predates the app's rename to Daily Tracker — not worth changing since it would force a migration for no benefit).
- `saveData(d)` is the single write path: stamps `updatedAtLocal`, writes localStorage, and fires `pushToCloud(d)`. New mutations should go through `saveData(DATA)`, not touch `localStorage` directly.
- Per-entry settings snapshots: calorie `entries[]` rows store the `activityKey`/`weeklyLossKg` that were active at save time, so changing Settings only affects new entries, never recalculates history. `calcForEntry()` reads from the entry first, falling back to current global settings only for legacy entries that predate this. Use the same snapshot-on-write approach for any future tool where "what were my settings on day X" matters.

### Cloud sync (optional, Firebase)

- Loaded via the Firebase **compat** SDK from CDN (`firebase-app/auth/firestore-compat.js`), not the modular SDK — kept as plain global-scope `<script>` tags since there's no bundler.
- `firebaseConfig` near the top of the script holds this project's real Firebase credentials. `firebaseConfigReady` requires every field to be filled in (not just `apiKey`) before `cloudSyncEnabled` is set — a half-configured project previously "enabled" the sign-in button while every attempt failed silently.
- Auth uses `signInWithPopup`, not `signInWithRedirect` — deliberately. Redirect-based sign-in relies on a cross-origin iframe between the app's actual hosting domain and Firebase's `authDomain`, which modern Chrome/Firefox/Safari block by default when the app isn't hosted on Firebase itself (this app is on GitHub Pages/Netlify). This was a real, debugged production issue; don't switch back without good reason.
- Sync model: one Firestore document per user (`users/{uid}`) holding the entire `DATA` blob. `syncOnSignIn()` does last-write-wins by comparing `updatedAtLocal` timestamps — not a field-level merge. Concurrent edits on two devices at the same moment will have one clobber the other.
- Everything cloud-related must degrade to a no-op if Firebase fails to load or isn't configured — never assume `cloudSyncEnabled`/`fbAuth`/`fbDb` are non-null without checking.

### Calculations worth knowing before touching them

- Calorie tool: BMR is Mifflin-St Jeor (`10×kg + 6.25×cm − 5×age + 5`); TDEE = BMR × activity multiplier (`EXERCISE_LEVELS`); daily calorie target = TDEE minus a deficit derived from the Settings weekly-loss slider (`kg/week × KCAL_PER_KG_FAT ÷ 7`), rounded to the nearest 10, clamped at `CALORIE_FLOOR` (1200 kcal). `AGGRESSIVE_LOSS_THRESHOLD_KG` (1.0 kg/week) drives the medical-supervision warning banner.
- Weight-trend / goal-date prediction (`computeTrend`, `goalDateFromTrend`) is a least-squares linear regression over every logged entry's (day-offset, weight) pair — deliberately not a naive first-vs-last calculation, so missed weigh-ins don't skew it.
- Press-Ups' target-date math is the opposite philosophy on purpose: a fixed, user-committed deadline (target count + days-to-achieve → resolved date), not a data-driven projection. Don't unify these two mechanics without checking with the user — they're intentionally different.
- Any date computed programmatically and stored/compared as a `YYYY-MM-DD` string must use `localDateStr()` (or the same local-component pattern as `todayStr()`) — never `.toISOString().slice(0,10)`. That was a real shipped bug: `toISOString()` formats in UTC, which silently lands on the wrong calendar day for anyone in a UTC+ timezone (this app's actual user is in the UK) whenever local midnight falls in the previous UTC day.

### Charts

Both tools render hand-rolled inline SVG line charts (`buildChartSVG` for calorie, `buildPressUpsChartSVG` for press-ups) — no charting library, to keep the app dependency-free and small. Both take a `range` key from `RANGE_OPTIONS` (7d/30d/1y/all) and scale the x-axis by real calendar days (via `daysBetweenDates`/`dateAddDays`), not by entry index, so date gaps (missed check-ins) show up honestly rather than being compressed away.

### PWA plumbing

`sw.js` precaches the app shell (`index.html`, `manifest.json`, icons) and serves cache-first with a network update-in-background for everything else, including cross-origin requests like the Firebase SDK (loaded with `crossorigin="anonymous"` specifically so the service worker can cache the response instead of getting an opaque, uncacheable one). Bump `CACHE_NAME` in `sw.js` if a change needs to force clients to drop stale cached assets.
