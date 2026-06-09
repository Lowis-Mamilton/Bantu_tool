# 班圖 Bantu — Claude 專案說明

## 專案概述

班圖（Bantu）是一個以純 Vanilla HTML/CSS/JavaScript 寫成的單檔 PWA 排班應用程式，目標裝置為 iPhone，支援加入主畫面離線使用。所有功能集中在 `index.html` 一個檔案中。

## 檔案結構

```
index.html      主程式（~1000 行，含所有 CSS/JS）
manifest.json   PWA manifest（名稱、icon、顯示模式）
sw.js           Service Worker（快取版本：bantu-v2）
icon/
  bantuicon.png 應用程式 icon（2362×2362 PNG）
```

## 資料結構

### Storage keys
- `bantu_shifts_v3` — 班次資料
- `bantu_cfg_v3` — 設定（工作、薪資、語言）

### Shifts 格式（v3）
```js
shifts["2026-06-07"] = [
  { id: "uuid", jobId: "j1", start: "09:00", end: "17:00", note: "" }
]
```

### Config 格式
```js
cfg = {
  lang: 'zh',           // 'zh' | 'en'
  defaultRate: 200,     // 預設時薪（當 job.hourlyRate 為 null 時使用）
  breakMinutes: 0,
  otRate: 1.34,
  stdHours: 8,
  jobs: [
    { id: 'j1', name: '工作1', color: '#3f9d72', hourlyRate: null },
  ]
}
```

## 核心架構

### i18n
- `LANGS.zh` / `LANGS.en` 兩個翻譯物件，含所有 UI 字串
- `L()` 回傳目前語言物件，`t(key)` 取特定字串
- `setLang(l)` 切換語言並重繪目前頁面
- `applyLang()` 更新所有帶有固定 ID 的靜態 DOM 元素

### 薪資計算
- `jobRate(job)` — 若 `job.hourlyRate != null` 使用個別時薪，否則用 `cfg.defaultRate`
- 加班計算：前 `stdHours` 小時按正常費率，超出部分乘以 `otRate`

### Modal 二層結構
- `#modal-day-view` — 顯示該日所有班次 + 新增按鈕
- `#modal-edit-view` — 新增/編輯單一班次表單
- 若當天無班次，`openDayModal()` 直接進入 edit view

### 班別自動分類（`autoClassLabel`）
- 開始時間 < 10:30 → 早班
- 10:30–14:29 → 中班
- >= 14:30 → 晚班

## 設計規範

### CSS 色彩 Token
```css
--bg: #faf8f4      暖米色背景
--card: #ffffff
--dark: #2a2620    深棕黑（today card、FAB、儲存按鈕）
--text: #2a2620
--t2: #6b655c      次要文字
--t3: #a8a299      輔助文字
--bd: #ece8e1      邊框
```

### 工作顏色盤（10色）
```js
const JOB_PALETTE = [
  '#3f9d72','#d9a23f','#6b62c4','#e05252','#0891b2',
  '#f97316','#8b5cf6','#ec4899','#2563eb','#0d9488'
]
```

## 注意事項

- 修改任何文字顯示時，**兩種語言（zh/en）都必須同步更新**，靜態 DOM 元素同時更新 `applyLang()`
- Service Worker 快取版本目前為 `bantu-v2`，每次更動快取策略時須升版
- 資料遷移函式 `migrateShifts()` 處理 v2（單物件/天）→ v3（陣列/天）的向下相容
- 不使用任何外部框架或 CDN，保持單檔可離線運作
