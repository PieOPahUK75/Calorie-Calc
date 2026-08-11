# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Daily Tracker — a single-file, installable PWA (no backend, no build step) hosting multiple small self-contained "tools" behind one shared shell. Currently: a Calorie/weight-loss Calculator, a Press-Ups tracker, and a Swimming tracker. Ships as static files served directly over HTTPS (GitHub Pages, Netlify, etc.) and installs to a phone home screen via "Add to Home Screen" or a PWABuilder-generated APK.

## Files

- `index.html` — the entire app: markup, CSS, and JS in one file (~2100 lines). Virtually all work happens here.
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

Since more tools are planned, the established pattern (used when Press-Ups and Swimming were added alongside the Calorie Calculator) is:

1. Add `{key, label, icon}` to `TOOLS`.
2. Add a `<div class="tool-panel" data-tool="yourkey">` inside each of the three `<section class="view">` blocks.
3. Give the tool its own namespaced slice of `DATA` (e.g. `DATA.yourTool = {settings:{...}, ...}`), with a `defaultYourTool()` factory and a **single migration function** (e.g. `migrateYourTool(data)`) called from all four places existing data can arrive from: `loadData()`, the remote-adopt branch inside `syncOnSignIn()`, `importData()`, and `clearAllData()` (via `defaultData()`). Factoring the migration into one function (see `migratePressUps`/`migrateSwimming`) rather than inlining the same checks four times is the current convention — follow it for the next tool too.
4. Write `renderYourToolToday/History/Settings()` functions and call them from `render()`, plus from `goTo('history')` if the tool has a History chart.
5. Reuse `RANGE_OPTIONS` + a `windowed...`-style helper (see `pressUpsWindowedDaily` or `swimWindowedSessions`) for any date-bucketed history/chart, rather than inventing new range logic.
6. Decide the record shape up front: **daily aggregate** (one row per calendar day, values accumulate through the day — Press-Ups' `sets[]` summed into `pressUpsDailyTotals()`) vs **discrete dated event** (one row per occurrence, multiple allowed per day, each with its own editable `date` field so past events can be backfilled — Swimming's `sessions[]`). Pick whichever matches how the activity is actually done; don't force one tool's shape onto a different kind of activity.

### Data & persistence

- A single `DATA` object holds everything (`settings` + `entries` for the calorie tool, `pressUps` for the press-up tool, `swimming` for the swim tracker), serialized as one blob to `localStorage` under `STORAGE_KEY = 'quickCalorieData_v1'` (the key name predates the app's rename to Daily Tracker — not worth changing since it would force a migration for no benefit).
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
- Weight-trend / goal-date prediction (`computeTrend`, `goalDateFromTrend`) is a least-squares linear regression over every logged entry's (day-offset, weight) pair — deliberately not a naive first-vs-last calculation, so missed weigh-ins don't skew it. This full-history trend still drives the goal-date projection text.
- The Today page's displayed "avg daily/weekly change" figures use a *different* trend: `computeRecentTrend(entries, 30)`, which fits the same regression but only over the trailing ~30 days (falling back to the full history if that window has fewer than 2 points). This exists because the all-time slope can lag noticeably behind a recent speed-up/plateau/slow-down — don't collapse these back into one trend without checking with the user, they're intentionally different views (long-term stability for the goal date, recent trajectory for "how am I doing lately").
- Press-Ups' target-date math is the opposite philosophy on purpose: a fixed, user-committed deadline (target count + days-to-achieve → resolved date), not a data-driven projection. Don't unify these two mechanics without checking with the user — they're intentionally different.
- Any date computed programmatically and stored/compared as a `YYYY-MM-DD` string must use `localDateStr()` (or the same local-component pattern as `todayStr()`) — never `.toISOString().slice(0,10)`. That was a real shipped bug: `toISOString()` formats in UTC, which silently lands on the wrong calendar day for anyone in a UTC+ timezone (this app's actual user is in the UK) whenever local midnight falls in the previous UTC day.
- Swimming only captures session **totals** (duration, distance, lengths, strokes), not a per-length breakdown, so its derived metrics (`swimSessionMetrics()`) are session averages, not true per-length figures: pace = seconds per 100m (`duration÷distance×100`), SWOLF = `(duration÷lengths)+(strokes÷lengths)`, stroke rate = `strokes÷(duration÷60)`. All three correctly return `null` (not `0` or `NaN`) when their inputs are missing (e.g. an open-water swim with no lengths/strokes) — preserve that when touching this code, the UI relies on `null` to render "–" instead of a misleading number.
- Swimming's pool length is intentionally a **per-session** editable field (prefilled from a Settings default, `defaultPoolLengthM`), not a fixed global — the user swims in different pools and occasionally open water, so a single locked-in pool length would actively get in the way. `wireSwimAutoDistance()` auto-fills distance from `lengths × poolLength` but backs off the moment the user types into the distance field directly (a `manual` flag in that closure), so open-water distance entry isn't clobbered.
- The calorie tool has a separate "weekly progress" line chart (`computeWeeklyBoundaries`/`computeWeeklyChanges`/`buildWeeklyChangeChart`) alongside the daily trend chart. It buckets your actual entries into Monday-anchored weeks and shows the change from each Monday to the next as a connected line (date-scaled x-axis, dots colour-coded green/red for loss/gain, `var(--muted)` line since the value itself swings both sides of zero) — but critically, the weight used for a Monday that has no exact weigh-in is **linearly interpolated** between the nearest logged entries either side (`estimatedWeightOnDate`), not carried forward or skipped. This was an explicit user requirement: "if I don't weigh in on a Monday, the value can still be determined." The very first boundary falls back to the first entry's raw weight if there's no earlier data to interpolate from (can't extrapolate backwards). Values outside the logged date range are never extrapolated — a boundary that would require it simply isn't produced. Has its own independent range state (`weeklyRange`) and its own trimmed range list `WEEKLY_RANGE_OPTIONS` (30D/1Y/All — no 7D, which would barely ever show more than one data point on a weekly-granularity chart), filtering the full weekly-change series down to the selected window rather than recomputing boundaries per range (keeps interpolation correct regardless of the window edges).

### Charts

All three tools render hand-rolled inline SVG line charts (`buildChartSVG` for calorie, `buildPressUpsChartSVG` for press-ups, `buildSwimmingChartSVG` for swimming) — no charting library, to keep the app dependency-free and small. All take a `range` key from `RANGE_OPTIONS` (7d/30d/1y/all) and scale the x-axis by real calendar days (via `daysBetweenDates`/`dateAddDays`), not by entry index, so date gaps (missed check-ins) show up honestly rather than being compressed away. Swimming additionally has a metric toggle (`SWIM_METRICS`: pace/SWOLF/stroke rate) above the range selector, since which metric best shows "improvement" is a user choice, not a fixed one — a pattern worth reusing if a future tool has more than one metric worth trending.
- The calorie chart's forecast/trend line only extends `domainEnd` forward for "all" — to the projected goal-crossing date (capped at a year) or a 30-day peek. This flip-flopped once already: 7D/30D/1Y briefly also extended forward (so the forecast wasn't "stuck in the past"), but that reserved roughly half the chart's width for the forecast on every rolling view, which the user then flagged as wasting space that should show the actual rolling window. Current answer: 7D/30D/1Y are rolling windows, `domainEnd` stays at "today", full width goes to the real data, and the dashed trend line on those ranges is a fit over the visible window only (not a forward projection) — only "all" gets a genuine forward-projecting forecast. If asked to make 30D/1Y forecast forward again, flag this tension explicitly rather than just re-adding it.
- The goal weight is only forced into the y-axis min/max for the "all" range. Forcing a possibly-distant goal weight into 7D/30D/1Y's scale made short-range views look artificially flat (a 0.5kg week's fluctuation compressed against a scale spanning all the way to a goal 15kg away). Short ranges now scale to the visible actual + forecast data only; the dashed goal line is still emitted but may simply render off the visible range when zoomed in, which is expected.
- The Today page has a "lowest weight" card (independent of whether a goal is set) computed as `entries.reduce(...)` finding the minimum `weightKg` — keep it goal-independent, it's a personal-best stat, not a goal-progress stat.
- The chart card has a full-screen expand button (`openChartFullscreen`/`closeChartFullscreen`) that reuses `buildChartSVG` at a larger rendered size inside a modal overlay (`#chartModalOverlay`) — it's the same SVG (viewBox-scaled), not a separate "big" chart builder. Currently only wired up for the calorie chart; extend to press-ups/swimming the same way if requested.
- All three line charts clamp `domainStart` to the first logged entry/session (`nominalStart > firstDate ? nominalStart : firstDate`) — a fixed 7/30/365-day lookback would otherwise leave dead blank space on the left for anyone who hasn't been using the app that long yet. `domainEnd` (today, plus any forecast extension) is untouched by this — only the left edge is clamped.
- All four charts (calorie daily, calorie weekly, press-ups, swimming) use a shared `dynamicChartWidth(totalSpanDays)` (`Math.max(480, totalSpanDays*4)`) for the SVG's `viewBox` width instead of a fixed 480. Height stays fixed and the `<svg>` gets `preserveAspectRatio="none"` plus inline `style="width:max(100%, {W}px);height:{H}px"` — this decouples X and Y scaling entirely, so short/rolling ranges still render at exactly 100% of the container (no distortion, no scrollbar) while long/dense ranges (mainly 1Y/All) render at their true pixel width and simply overflow their wrapper. The wrapper divs (`#chartWrap`, `#chartModalWrap`, `#puChartWrap`, `#swChartWrap`, `#weeklyChartWrap`) have `overflow-x:auto` for exactly this reason — don't remove it, dense charts rely on it to become scrollable instead of squashed. Don't reintroduce `width:100%;height:auto` on `svg.chart` globally; that's what caused the original squashing (aspect-ratio-locked scaling forces height down as width per-point shrinks).
- All four charts also get evenly-spaced X-axis date labels via the shared `buildXAxisLabels(domainStart, domainEnd, xAt, W, padL, padR, y)` helper (up to 5 ticks, spaced by fraction of the domain so it works for any span, tick count shrinks on narrow charts via `Math.floor(availableW/70)+1` so labels never overlap). `padB` was bumped from 24 to 34 (30 on the weekly chart) across all four builders to make room for this row without colliding with the existing Y-axis bottom value label.

### PWA plumbing

`sw.js` precaches the app shell (`index.html`, `manifest.json`, icons) and serves cache-first with a network update-in-background for everything else, including cross-origin requests like the Firebase SDK (loaded with `crossorigin="anonymous"` specifically so the service worker can cache the response instead of getting an opaque, uncacheable one).

**Bump `CACHE_NAME` in `sw.js` on every change to `index.html`, `manifest.json`, or the icons — not just "when it seems needed."** Cache-first means an installed app can silently keep serving a stale `index.html` indefinitely; this bit in practice (a duration-field fix appeared to "do nothing" purely because the cache name hadn't changed). Even after bumping it, the user may need to fully close and reopen the installed app twice — once to pick up the new service worker and trigger a background refetch, once more to see the refreshed content — since `activate` only clears old cache entries, it doesn't force an immediate repaint of an already-open page.
