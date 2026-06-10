# Bantu (班圖) — Claude Project Notes

## Overview

Bantu is a single-file PWA shift-scheduling app written in pure vanilla HTML/CSS/JavaScript, targeting iPhone with offline home-screen support. All functionality lives in one file: `index.html`.

## File Structure

```
index.html      Main app (~1100+ lines, all CSS/JS included)
manifest.json   PWA manifest (name, icon, display mode)
sw.js           Service worker (cache version: bantu-v4)
robots.txt      SEO crawler rules (points to sitemap.xml)
sitemap.xml     SEO sitemap (single page)
icon/
  bantuicon.png App icon (2362×2362 PNG)
```

## Data Structures

### Storage keys
- `bantu_shifts_v3` — shift data
- `bantu_cfg_v3` — config (jobs, pay, language, templates)

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
- Service worker cache version is currently `bantu-v4` — bump it whenever `index.html` (or other cached assets) changes, so installed PWA users get the update
- `migrateShifts()` handles backward compatibility from v2 (single object per day) to v3 (array per day)
- No external frameworks or CDNs — keep this a single offline-capable file
