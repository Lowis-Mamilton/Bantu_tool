# Bantu (班圖) — Claude Project Notes

## Overview

Bantu is a single-file PWA shift-scheduling app written in pure vanilla HTML/CSS/JavaScript, targeting iPhone with offline home-screen support. All functionality lives in one file: `index.html`.

## Release Rules — non-negotiable

Every change ships to people who already have Bantu on their home screen with months of real shift data in it. Two things MUST hold for every single update, no exceptions:

### 1. An update must never make existing users' data disappear

All user data lives only in `localStorage` (`bantu_shifts_v3` / `bantu_cfg_v3`). There is no backend and no backup — a release that eats someone's schedule is unrecoverable for them.

- **Never rename or repurpose a storage key** without a migration that reads the old key and writes the new one. Keep old migrations in place permanently — a user may update from any old version, skipping many releases in between
- **Never clear or blindly overwrite storage** outside an explicit user action. Only three places may replace bulk data — `btn-clear`, `applyImport()` and `btn-restore` — and every one of them calls `backupNow()` first ([index.html:1391](index.html#L1391))
- **Schema changes must be additive and tolerate missing fields.** Read with a fallback (`s.note||''`, `cfg.x ?? DEFAULT_CFG.x`); never assume a field exists just because the new code writes it. Give new shift fields a default at read time instead of rewriting stored data
- **New `cfg` fields go in `DEFAULT_CFG`,** and normalisation belongs in `normalizeCfg()` ([index.html:799](index.html#L799)) so load and import stay in sync. Its `{...DEFAULT_CFG, ...raw}` is a **shallow** merge — a new field added inside `cfg.jobs[]` or `cfg.templates[]` will NOT appear on a returning user's stored objects, so those must be read defensively too
- **Structural changes follow the `migrateShifts()` pattern** ([index.html:786](index.html#L786)): accept the old shape, convert it to the new one, and run it on load. Convert old entries — don't discard what you don't recognize
- **Test the upgrade path with real old data**, not just an empty `localStorage`: seed the previous version's keys, load the new build, and confirm every shift, job, template and setting survives

### 2. Bump the service worker cache version on every release

`sw.js` serves cached assets. If `CACHE` keeps the same value, installed users keep the old cached `index.html` forever and would have to delete and re-add the app to see any update.

- Bump `CACHE` in [sw.js](sw.js) (currently `bantu-v6`) on **every** change to `index.html`, `manifest.json` or the icon — even a one-character text fix
- Also update the version noted in this file (File Structure section and Notes)
- The `install`/`activate` handlers already call `skipWaiting()` + `clients.claim()` and delete stale caches, so a bumped version reaches users on their next launch with **no reinstall**. Don't remove that behaviour
- Any newly cached file must be added to `ASSETS` in `sw.js`

## File Structure

```
index.html      Main app (~1100+ lines, all CSS/JS included)
manifest.json   PWA manifest (name, icon, display mode)
sw.js           Service worker (cache version: bantu-v6)
robots.txt      SEO crawler rules (points to sitemap.xml)
sitemap.xml     SEO sitemap (single page)
icon/
  bantuicon.png App icon (2362×2362 PNG)
```

## Data Structures

### Storage keys
- `bantu_shifts_v3` — shift data
- `bantu_cfg_v3` — config (jobs, pay, language, templates)
- `bantu_shifts_v3_bak` / `bantu_cfg_v3_bak` — one-slot undo snapshot, written by `backupNow()` before any bulk overwrite (import, clear, restore)
- `bantu_install_dismissed` — install-banner dismissal flag

### Shifts format (v3)
```js
shifts["2026-06-07"] = [
  { id: "uuid", jobId: "j1", start: "09:00", end: "17:00", note: "" }
]
```

### Config format
```js
cfg = {
  lang: 'zh',           // 'zh' | 'en'
  currency: 'TWD',      // currency code, one of CURRENCIES (10 options)
  defaultRate: 200,     // default hourly rate (used when job.hourlyRate is null)
  breakMinutes: 0,
  otRate: 1.34,
  stdHours: 8,
  deduction: 758,       // fixed monthly deduction subtracted from estimated salary
  jobs: [
    { id: 'j1', name: '工作1', color: '#3f9d72', hourlyRate: null },
  ],
  templates: [
    { id: 'tpl1', name: '早班', jobId: 'j1', start: '09:00', end: '17:00', note: '' }
  ]
}
```

## Core Architecture

### i18n
- `LANGS.zh` / `LANGS.en` — translation objects containing all UI strings
- `L()` returns the current language object, `t(key)` returns a specific string
- `setLang(l)` switches language and re-renders the current page
- `applyLang()` updates all static DOM elements that have fixed IDs

### Salary calculation
- `jobRate(job)` — uses `job.hourlyRate` if set, otherwise falls back to `cfg.defaultRate`
- Overtime: hours up to `stdHours` are paid at the normal rate; hours beyond that are multiplied by `otRate`
- `cfg.deduction` is subtracted from the monthly total in the Stats page (net salary, floored at 0)

### Currency
- `CURRENCIES` — fixed list of 10 common currencies (`{code, symbol, zh, en}`), selectable in Settings
- `curSym()` returns the symbol for `cfg.currency`, used everywhere a price is displayed (stats, job hourly-rate tags, settings units)

### Monthly goal progress
- The goal-progress card on the Stats page (`#goal-card`) is fully derived from the user's own shift schedule — there is no manual goal setting
- `goal` = total scheduled hours for the displayed month (sum of all shift hours); `worked` = sum of hours for shifts on or before today (`k<=tKey()`)
- Shown only when `goal > 0`; displays a progress bar (`worked / goal`), percentage, and either remaining hours (future shifts this month) or a "goal reached" message

### Two-layer modal
- `#modal-day-view` — lists all shifts for the selected day + "Add Shift" button
- `#modal-edit-view` — form for adding/editing a single shift
- If the day has no shifts, `openDayModal()` jumps straight to the edit view

### Day navigation inside the modal
- Edit view bottom row: `navDay(dir)` — "save and continue on the next/prev day". It calls `commitShift()` first (only when `formDirty`), so a filled-in form is written before the date moves; a validation failure aborts the navigation
- `formDirty` is set by any field change, and pre-set to `true` when the form was seeded from an existing shift or from `chainEntry`. An untouched blank day (just the 09:00/18:00 defaults) navigates without creating a phantom shift
- `chainEntry` = `{jobId, start, end}` of the last committed shift; `openEditView(null)` uses it as the defaults so consecutive days can be filled by tapping the same button. Cleared in `openDayModal()` and `closeModal()`, so carry-over only lives inside one continuous chain
- Day view bottom row: `jumpDay(dir)` — plain date navigation, no saving (nothing unsaved exists there)
- Both rows' labels are dynamic (they show the target date) and are refreshed by `renderDayNavLabels()`, called from `showDayView()`, `openEditView()` and `applyLang()`

### Data import / export (Settings → 資料)
- `btn-export` writes `{shifts, cfg}` as JSON; `btn-import` reads one back via a hidden `#imp-file` input + `FileReader`
- `parseImport()` accepts a full `{shifts, cfg}` export **or** a bare day-map, then `sanitizeImported()` ([index.html:1422](index.html#L1422)) keeps only `YYYY-MM-DD` keys with valid `HH:MM` times, accepts v2 single-object days, fills missing `id`/`jobId`, and **spreads each shift so unknown fields survive**. Rejected entries are counted and reported, never silently dropped
- `renderImportSummary()` shows file vs. current counts, conflicting days and what the file's `cfg` contains, then offers **merge** (`{...shifts, ...file}`, file wins per day) or **replace** (file only, behind a second `confirm`)
- `applyImport()` always calls `backupNow()` first; `#btn-restore` swaps the live data and the backup, so undo is itself undoable. The restore row is hidden until a backup exists (`hasBackup()` in `renderSettings()`)
- An imported `cfg` goes through `normalizeCfg()` — never trust a file's currency/lang/jobs

### Shift templates (固定班次)
- Managed in Settings under "Shift Templates" — each template stores name, jobId, start, end, note
- In the edit modal, a "Quick Apply" chip row (`renderQuickApply()`) appears above the job picker when templates exist; tapping a chip fills in job/start/end/note via `applyTemplate()`

### Auto shift classification (`autoClassIndex` / `autoClassLabel`)
- `autoClassIndex(start)` returns 0/1/2; `autoClassLabel(start)` maps that to `L().cat[i]`
- Start time < 10:30 → Morning (早班, index 0)
- 10:30–14:29 → Afternoon (中班, index 1)
- >= 14:30 → Evening (晚班, index 2)

### PWA install banner
- `#install-banner` on the Today page tells users the app can be installed to their home screen
- Hidden if already running standalone (`isStandalone()`, checks `display-mode: standalone` / `navigator.standalone`) or if previously dismissed (`localStorage['bantu_install_dismissed']`)
- On iOS, shows static "tap Share → Add to Home Screen" instructions (no install button, since iOS Safari has no `beforeinstallprompt`)
- On Android/Chrome, `beforeinstallprompt` is captured into `deferredInstallPrompt`; the banner shows an "Install" button that calls `installApp()` (`deferredInstallPrompt.prompt()`)
- `renderInstallBanner()` is called from `applyLang()` so banner text updates with language switches

### SEO
- `<head>` includes `meta description`/`keywords`/`robots`, OG/Twitter tags, `rel=canonical`, and a `WebApplication` JSON-LD block — all pointing at the deployed URL `https://bantu.bantutw.workers.dev/`
- `robots.txt` and `sitemap.xml` at the repo root reference the same canonical URL
- If the deployment URL ever changes, update all of: canonical link, OG/Twitter `*:url`/`*:image`, JSON-LD `url`/`image`, `robots.txt`, and `sitemap.xml`

### Stats page charts
- `computeMonthStats(y, m)` aggregates one month's shifts into `{totalH, workedH, sc, salary, byJob, days, catCounts, wdHours}`
  - `catCounts` — shift counts per `autoClassIndex` bucket (早/中/晚班), used by `renderCatBreakdown()` (`#cat-breakdown`, `CAT_COLORS`)
  - `wdHours` — total hours per weekday (`Date.getDay()`, 0=Sun), used by `renderWeekdayChart()` (`#wd-chart`); Sunday/Saturday bars use the same red/blue tint as the calendar weekday headers
- `renderStats()` calls `computeMonthStats()` for the displayed month and for the previous month (with year wraparound), then calls `renderMomComparison(cur, prev)` to populate `#mom-hours-val` / `#mom-salary-val` (`.mom-pos/.mom-neg/.mom-flat`); shows `—` when the previous month has no shifts

## Design Specs

### CSS color tokens
```css
--bg: #faf8f4      warm cream background
--card: #ffffff
--dark: #2a2620    dark brown-black (today card, FAB, save buttons)
--text: #2a2620
--t2: #6b655c      secondary text
--t3: #a8a299      tertiary text
--bd: #ece8e1      border color
```

### Job color palette (10 colors)
```js
const JOB_PALETTE = [
  '#3f9d72','#d9a23f','#6b62c4','#e05252','#0891b2',
  '#f97316','#8b5cf6','#ec4899','#2563eb','#0d9488'
]
```

## Notes

- When changing any displayed text, **both languages (zh/en) must be updated together**, and static DOM elements must also be updated in `applyLang()`
- Service worker cache version is currently `bantu-v6` — see **Release Rules** above; it must be bumped on every release
- `migrateShifts()` handles backward compatibility from v2 (single object per day) to v3 (array per day) — keep it, and add to it rather than replacing it
- No external frameworks or CDNs — keep this a single offline-capable file
