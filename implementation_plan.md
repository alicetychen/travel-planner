# 旅遊規劃系統 v2.15 行程時間精準對齊與單點時間接駁修復計畫

## 1. 問題分析與痛點確認
- **使用者回報問題**：「前一項結束時間+移動時間和下一項開始時間對不起來」（截圖：前項為單點時間 `04:20` 機場專車接送出發，接駁移動自訂 45 分鐘，接駁條顯示 `(05:20 ➔ 06:05)`，次項班機開始時間為 `06:05`）。
- **根本原因 (Root Cause)**：
  1. `parseTimeString` 解析單點時間（如 `04:20`）時，系統預設為其加上 60 分鐘，導致回傳之 `endMinutes` 為 `05:20`。
  2. 接駁推算與連動調整依賴 `pPrev.endMinutes`，以 `05:20` 作為起算點加上 45 分鐘車程，使得次項班機行程被推算為 `06:05` 開始，造成平白多出 60 分鐘差距，時間完全對不攏。
- **目標**：
  1. 徹底修正單點時間之結束時間與停留時長，單點時間結束時間即開始時間，停留時長為 0 分鐘。
  2. 接駁起算時間依據「前項有效結束時間」（單點時間即其出發時間，區間時間即其結束時間）。
  3. 時間軸接駁條清楚標註移動起迄時間，並新增「⚡ 一鍵對齊次站」快捷按鈕，讓時程未吻合時能秒級自動校正。
  4. 接駁設定彈窗、停留時間調整彈窗、即時影響分析全方位同步校準。
  5. 升級至 v2.15，通過 DOM 結構驗證與單元測試，完成 Git 版控與 GitHub 部署。

---

## 2. 預計修改項目與架構規劃

### A. 時間解析演算法 (`parseTimeString`)
- 當匹配單點時間（如 `04:20`、`08:30 以後`）時：
  - `startMinutes`: `startM`
  - `endMinutes`: `startM`
  - `durationMinutes`: `0`
  - `startTimeStr`: `HH:MM`
  - `endTimeStr`: `HH:MM`
  - `hasEnd`: `false`

### B. 接駁時間計算與時間軸呈現 (`renderTimelineView`)
- 前項起算時間：`prevEndM = prevParsed.hasEnd ? prevParsed.endMinutes : prevParsed.startMinutes`
- 前項時間標籤：`prevTimeLabel = prevParsed.hasEnd ? prevParsed.endTimeStr : prevParsed.startTimeStr`
- 自訂車程預期抵達時間：`expNextStartM = prevEndM + displayMinutes`
- 若次項開始時間與預期抵達時間不符（如曾被舊版推算算錯），接駁條顯示 `(${prevTimeLabel} ➔ 預計 ${expNextTimeStr})`，並在接駁動作區提供「⚡ 對齊次站 (${expNextTimeStr})」快捷按鈕。
- 若兩站時間已完美吻合，顯示 `(${prevTimeLabel} ➔ ${parsed.startTimeStr})`。

### C. 新增一鍵對齊次站函數 (`syncNextEventWithTransit`)
- 點擊後以「前項有效結束時間 ＋ 移動時間」精準平移次站開始時間，保持次站原本停留時長，並連鎖順延後續行程。

### D. 接駁設定彈窗與即時預覽推算 (`openTransitSettingsModal`, `updateTransitLivePreview`, `handleSaveTransitSubmit`)
- 起算點統一採用前項有效結束時間，提示文字顯示「前站於 04:20 出發 ➔ 移動車程 45 分鐘 ➔ 次站時間將自動調整為 05:05」。
- 儲存時以 `prevEndM + minutes` 精準平移次站開始時間。

### E. 景點停留時間與行程編輯防護
- 停留時間修改中，舊結束時間取用 `oldEndM = parsed.hasEnd ? parsed.endMinutes : parsed.startMinutes`。
- 行程編輯時保留現有之 `transit` 物件設定。

### F. 版本升級與同步
- 升級版本號至 `v2.15`。
- 同步至 `travel_planner.html`。
- 透過 `uv run python` 進行自動化測試與 DOM 檢驗。
- Git Commit 並 Push 到遠端倉庫。
