# 旅遊規劃 v2.18 實作計畫 (Implementation Plan)

## 1. 現象與問題核心診斷

### 問題 A：每次開啟同行人數自動變成 4 人、標題修改為「重慶自由行」跑掉
- **根因**：
  在 `index.html` 第 5112 行：
  ```javascript
  if (!appState.plans['plan-chongqing'] || !appState.plans['plan-chongqing'].days || appState.plans['plan-chongqing'].days.length < 7 || appState.plans['plan-chongqing'].version !== 2026100503) {
      appState.plans['plan-chongqing'] = JSON.parse(JSON.stringify(PRESET_CHONGQING));
      appState.activePlanId = 'plan-chongqing';
      saveState();
  }
  ```
  `PRESET_CHONGQING.version` 為 `2026100504`，導致 `version !== 2026100503` 在每次頁面載入（`loadState`）時永遠為真，強制將使用者修改後的重慶計畫覆蓋回範本預設值（4 人、標題為「重慶 7 天 6 夜自由行 🇨🇳」），並推送至雲端及本地 localStorage。
- **解法**：
  移除版本死邏輯覆蓋。僅在 `!appState.plans['plan-chongqing']` 時才初始化預設範本；一旦使用者自訂過，完整尊重使用者資料。

---

## 2. 功能設計與技術架構

### 功能 B：交通與住宿費用計算模組
#### 1. 交通（機票大交通）費用資料結構：
```javascript
plan.flights = {
    // 既有欄位: pnr, categoryTag, isTransfer, legs...
    cost: 2500,               // 預估費用數值
    actualCost: 2450,         // 實際支出數值 (可選)
    costMode: 'single',       // 'single': 每人單價 / 'total': 全團總額
    costCurr: 'LOCAL',        // 'LOCAL' (當地幣 CNY 等) 或 'TWD'
    outboundCost: 1250,       // 去程分段費用 (可選)
    inboundCost: 1250         // 回程分段費用 (可選)
}
```
- **計算公式**：
  - 若為 `single`：每人費用 = `cost`；全團總額 = `cost × memberCount`。
  - 若為 `total`：全團總額 = `cost`；每人費用 = `cost / memberCount`。
  - 幣別自動依照計畫匯率（`curr.rate`）雙向換算當地幣與新台幣（NT$）。
- **UI 呈現**：
  - `editFlightModal`：提供費用輸入區塊（費用金額、計費模式每人/全團、幣別切換、實付金額）。
  - `sideFlightCardsContainer`：在側邊欄交通卡片底部呈現「💰 交通費用計算」精緻徽章與金額說明。

#### 2. 住宿（各飯店房費）費用資料結構：
```javascript
hotelItem = {
    // 既有欄位: name, city, stayLabel, checkIn, checkOut, nights, addr, phone, notes...
    priceType: 'perNight',    // 'perNight': 每晚房價 / 'total': 整段總價
    price: 680,               // 金額 (每晚房價或整段總價)
    actualCost: 2600,         // 實際支付總額 (可選)
    rooms: 1,                 // 房間數量 (預設 1 間)
    costCurr: 'LOCAL'         // 'LOCAL' 或 'TWD'
}
```
- **計算公式**：
  - 若 `priceType === 'perNight'`：該飯店總房費 = `price × nights × rooms`；每人均攤 = 總房費 / `memberCount`。
  - 若 `priceType === 'total'`：總房費 = `price × rooms`；每人均攤 = 總房費 / `memberCount`。
- **UI 呈現**：
  - `editHotelModal`：提供每晚價格/整段總價輸入、房間數（可調）、即時試算總額與每人分攤預覽。
  - `sideHotelCardsContainer`：在側邊欄住宿卡片中展示「💰 房費計算」區塊（每晚、晚數、房間數、總房費、每人分攤及折合台幣）。

---

### 功能 C：全新「實際費用與行程計算中心」 (Actual Expenses Tracker)
1. **導覽分頁擴充**：
   - 在行程上方分頁列增加按鈕：`🧾 實際花費與行程記帳 (Actual Expenses)`。
   - 點擊後切換至全新視圖 `#view-actual-expenses`。
2. **行程活動關聯記帳**：
   - 每個行程活動（`event`）支援欄位：
     - `actualCost`：實際支付金額（預設為空或與預估相同）。
     - `isPaid`：布林值，標記是否已結帳/已支付。
     - `paidBy`：支付人/分攤備註（可選）。
   - 在行程時間軸（Timeline）中活動項目增加「實際支出」快捷標籤與編輯捷徑。
3. **實際費用計算儀表板（KPI & 統計分析）**：
   - **核心 KPI**：
     - 總預估預算 vs 總實際支出（單人與全團）。
     - 差額分析：結餘（綠色，低於預算）或超支（紅色警示，超過預算）。
     - 記帳完成進度條（已記帳筆數 / 總支出項目數）。
   - **分項支出對比長條圖與圓餅清單**：
     - ✈️ 機票大交通（預估 vs 實付）
     - 🏨 飯店住宿（預估 vs 實付）
     - 🍽️ 餐飲開銷（預估 vs 實付）
     - 🚗 市區交通/包車（預估 vs 實付）
     - 🎫 景點門票（預估 vs 實付）
     - 🛍️ 購物與自訂其他支出
   - **每日明細記帳表格**：
     - 清晰列出當日各行程項目、預估金額、實付金額、差額與記帳狀態，支援直接內聯輸入或彈窗快捷更新。

---

## 3. 實施步驟
1. 修正 `loadState()` 覆蓋 Bug，確保使用者修改的同行人數與標題永久保存。
2. 擴充 `editFlightModal` 與 `renderSidebarFlights`，加入交通費用計算。
3. 擴充 `editHotelModal` 與 `renderSidebarHotel`，加入住宿費用動態試算。
4. 升級 `calculatePlanBudgetStats` 與 `renderTripSummary`，納入機票與住宿總額。
5. 建立 `view-actual-expenses` 視圖與渲染引擎 `renderActualExpensesView()`。
6. 在活動彈窗中支援填寫實際花費與已付標記。
7. 透過 Python 自動化測試（`uv run python`）驗證所有計算邏輯與 DOM 結構。
8. 升級至 v2.18 並同步 `travel_planner.html` 與 Git 提交。
