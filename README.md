# Bantu (班圖)

> A PWA for shift scheduling, hours tracking, and salary estimation

![Bantu Icon](./icon/bantuicon.png)

## Features

- **Calendar view** — see each day's shifts at a glance, auto-colored by morning/afternoon/evening shift
- **Multi-job support** — add multiple jobs (Job 1, Job 2…), with multiple shifts per day
- **Salary estimation** — independent hourly rate per job, automatic overtime calculation
- **Shift templates** — save frequently-used shift presets and apply them with one tap
- **Monthly deduction** — subtract a fixed monthly amount (e.g. fees) from the estimated salary
- **Stats reports** — monthly total hours, days worked, shift count, per-job breakdown
- **Chinese / English toggle** — full UI localization
- **PWA** — installable to iPhone/Android home screen, works offline

## Usage

Open `index.html` directly in a browser, or deploy to a static server over HTTPS to install as a PWA.

### iPhone install steps
1. Open the URL in Safari
2. Tap the "Share" button
3. Select "Add to Home Screen"

## Project Structure

```
Bantu_tool/
├── index.html      # Main app (single-file PWA, pure HTML/CSS/JS)
├── manifest.json   # PWA manifest
├── sw.js           # Service worker (offline cache)
├── icon/
│   └── bantuicon.png
└── README.md
```

## Data Storage

All data is stored in the browser's `localStorage` — nothing is uploaded to any server. You can export a JSON backup from the Settings page.

## Development

No framework dependencies. Pure vanilla HTML/CSS/JavaScript, single-file architecture.
