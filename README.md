# 班圖 Bantu

> 排班紀錄、時數統計與薪資估算的 PWA 應用程式

![班圖 Bantu Icon](./icon/bantuicon.png)

## 功能

- **月曆檢視** — 一眼看出每天的班次與時段，自動標色區分早/中/晚班
- **多工作支援** — 可新增多份工作（工作1、工作2…），一天可同時有多個班次
- **薪資估算** — 支援每份工作獨立時薪設定，自動計算加班費
- **統計報表** — 月份總工時、出勤天數、班次數、各工作分項統計
- **中英文切換** — 介面支援繁體中文 / English
- **PWA** — 可安裝至 iPhone/Android 主畫面，離線使用

## 使用方式

直接用瀏覽器開啟 `index.html`，或部署至靜態伺服器後以 HTTPS 訪問即可安裝為 PWA。

### iPhone 安裝步驟
1. 用 Safari 開啟網址
2. 點底部「分享」按鈕
3. 選「加入主畫面」

## 專案結構

```
Bantu_tool/
├── index.html      # 主程式（單檔 PWA，純 HTML/CSS/JS）
├── manifest.json   # PWA Manifest
├── sw.js           # Service Worker（離線快取）
├── icon/
│   └── bantuicon.png
└── README.md
```

## 資料儲存

所有資料儲存於瀏覽器 `localStorage`，不上傳任何伺服器。可於設定頁匯出 JSON 備份。

## 開發

無任何框架依賴，純 Vanilla HTML/CSS/JavaScript 單檔架構。
