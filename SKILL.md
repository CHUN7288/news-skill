---
name: news
description: 抓取外匯經濟日曆，篩選對 XAUUSD（黃金）高影響力的事件並依時間列出。使用 /news 觸發（預設今日全天）、/news today（今日 06:00–22:00 GMT+8）、/news tomorrow（明日 04:00–22:00 GMT+8）。當用戶輸入 /news 系列指令時立即執行，無需額外說明或確認。資料來源為 ForexFactory JSON API 與 Myfxbook，自動套用 XAUUSD 篩選邏輯。
---

# /news — XAUUSD 高影響力經濟日曆

## 呼叫模式與時間範圍

| 指令 | 目標日期 | GMT+8 時間範圍 |
|------|----------|----------------|
| `/news` | 今日 | 全天（無限制） |
| `/news today` | 今日 | 06:00 – 22:00 |
| `/news tomorrow` | 明日 | 04:00 – 22:00 |

- **目標日期**：決定從哪一天篩選事件（ForexFactory `date` 欄位；Myfxbook 日期區塊）
- **GMT+8 時間範圍**：時區轉換後才套用，在輸出前過濾掉範圍外的事件
- `/news`（無參數）：維持現有行為，不套用時間範圍限制

從 **ForexFactory 與 Myfxbook 兩個來源** 各自抓取目標日期的經濟事件，兩者皆必須查詢，套用篩選後**合併去重，統整成一張表格**輸出。

## 資料來源（兩者皆必查，缺一不可）

### 來源一：ForexFactory JSON API（WebFetch）
```
https://nfs.faireconomy.media/ff_calendar_thisweek.json
```
回傳本週所有事件，篩選 `date` 欄位等於**目標日期**（`/news` 與 `/news today` 為今日；`/news tomorrow` 為明日）。欄位對照：
- `date` — 事件日期
- `time` — 時間（ET，格式如 `8:30am`）
- `currency` — 貨幣代碼（USD、EUR 等）
- `impact` — 影響力：`High`、`Medium`、`Low`、`Holiday`
- `title` — 事件名稱
- `forecast` / `previous` — 預期值 / 前值（不顯示）

### 來源二：Myfxbook（Playwright，必查）
```
https://www.myfxbook.com/forex-economic-calendar
```
用 Playwright 開啟，提取**目標日期**的事件後套用篩選。
- `/news` 與 `/news today`：提取今日事件
- `/news tomorrow`：提取明日事件（識別明日日期的區塊）

**存取注意事項：**
- Myfxbook 封鎖 WebFetch（回傳 403），**必須使用 Playwright**
- 若 Playwright 未安裝，在輸出中明確標示「Myfxbook：無法存取（需安裝 Playwright）」，不可靜默略過
- Myfxbook 時間已為 **GMT+8**，無需額外轉換
- **⚠️ KRW 等非主要貨幣缺失時的修復方法（一次性設定）：**
  1. 點擊 `#filterButton`（Filter 圖示）
  2. 在貨幣清單中勾選 KRW（或點 "All" 全選）
  3. 用 Playwright native click 點擊 `#calendar-settings-apply`（JS `.click()` 無效，必須用 `page.locator('#calendar-settings-apply').click()`）
  4. 等待 4 秒讓頁面重載
  5. 設定會永久保存在 Playwright browser session 中，之後每次 navigate 都自動包含 KRW

## 篩選邏輯

### A. 所有貨幣 — 關鍵字匹配（演講類除外）
事件名稱（`title`，不分大小寫）含以下任一關鍵字，**不論貨幣、不論影響力**，一律納入：

| 關鍵字 |
|--------|
| interest rate |
| press conference |
| CPI |
| inflation |
| GDP |
| growth rate |
| PCE |
| PMI |
| unemployment |
| employment |
| employ |
| unemploy |

### A2. 演講類（speech / speaks）篩選規則
演講類事件（事件名稱含 `speech` 或 `speaks`，不分大小寫）適用以下專屬規則，**不套用 A 的一般邏輯**：

| 貨幣 | 納入條件 |
|------|----------|
| USD | 所有演講，不論影響力 |
| AUD、CAD、EUR、GBP | 僅 `impact == "High"` 的演講 |
| 其他貨幣（CHF、JPY 等） | 一律排除 |

### A-USD. USD 專屬關鍵字（演講類除外）
**currency == USD** 且事件名稱含以下任一關鍵字（不分大小寫），**不論影響力**，一律納入。  
此規則僅限 USD，其他貨幣不適用。

| 關鍵字 | 備注 |
|--------|------|
| FED | 涵蓋 Fed、Fed's 等 |
| federal | 涵蓋 Federal Reserve 等 |
| ISM | |
| PPI | |
| Balance of trade | |
| Jobless | 涵蓋 Jobless Claims 等 |
| Consumer confidence | |
| Consumer sentiment | |
| JOLTs | 不分大小寫 |
| Job opening | 涵蓋 Job Openings 等 |
| Average Hourly Earnings | |
| 10Y Note Auction | |
| 30Y bond Auction | |
| New home | 涵蓋 New Home Sales 等 |
| Goods orders | 涵蓋 Durable Goods Orders 等 |
| manufacture | **字根**，涵蓋 Manufacturing、Manufactured 等所有派生詞 |
| Retail sales | |
| PCE | 已在 Rule A 全貨幣，此處重複以示強調 |

> 演講類（含 speech / speaks）的 USD 事件已由 Rule A2 全納，不需此規則再處理。

### B. 所有貨幣 — 高影響力全納入
**所有貨幣**（不限五大貨幣）中，`impact == "High"` 的事件，不論名稱為何，一律納入。

> ⚠️ 注意：舊版規則 B 只限 AUD/CAD/EUR/GBP/USD，已修正為**全貨幣適用**。

## 輸出格式規則

### 1. 同時間事件：只顯示一條代表，演講全列
相同時間有多個符合條件的事件時，**只顯示其中一條作為代表**（任選），其餘省略。
**例外：演講類事件（title 含 `speech` 或 `speaks`）不受此限，所有符合 A2 規則的演講都必須完整列出。**

時間欄位只填一次，後續同時間事件留空對齊。

**示範（17:00 有三個事件，只留一條代表；演講全列）：**
```
| 17:00 | EUR-Inflation Rate YoY (Apr)        | Low  |
| 20:30 | ***USD-FOMC Member Cook Speaks      | Low  |
|       | ***USD-FOMC Member Daly Speaks      | Low  |
```

### 2. All Day 全天事件置頂、Tentative 事件次之
- 符合篩選規則的全天事件，時間欄位標示為 `All Day`，一律列在表格**最上方**。
- 符合篩選規則的 Tentative 事件，時間欄位標示為 `Tent.(時間或找不到)`，排在所有 `All Day` 事件之後、有具體時間的事件之前。若無 `All Day` 事件，則 `Tent.` 事件置於表格最上方。
  - **Tentative 實際時間查詢（必做）：** 對每個 Tentative 事件，用 WebSearch 搜尋該事件的慣常發布時間（如 "Cleveland Fed Inflation Expectations release time"）。若找到可信結果（官方網站、彭博、路透等），將 GMT+8 時間填入括號，格式為 `Tent.(HH:MM)`；若搜尋後確實找不到，標示為 `Tent.(找不到)`。不可猜測或沿用 ForexFactory JSON 的佔位時間戳。

**排列順序：**
```
All Day 事件（若有）
Tent. 事件（若有）
有具體時間的事件（依 GMT+8 時間升冪排列）
```

### 3. 演講類事件名稱前加 `***`
符合 A2 演講規則而納入的事件，在事件名稱前加上 `***`。

**示範：**
```
| 15:05 | ***EUR-ECB de Guindos Speech        | High |
| 17:45 | ***USD-FOMC Member Cook Speaks      | Low  |
```

### 其他格式規定
- 時間為 Tentative → 標示 `Tent.(HH:MM)` 或 `Tent.(找不到)`（查詢規則見§2）
- Holiday 全天事件 → 若名稱不含篩選關鍵字且非 High impact，略過不顯示
- 換算後超過 24:00（隔日台北時間）→ 標示 `(+1)`
- 預期值與前值**不顯示**
- 表格後**不加**今日重點摘要或任何總結

## 執行步驟

**步驟 0：判斷模式**
- 無參數 → `mode=default`，目標日期 = 今日，不套用時間範圍過濾
- 參數為 `today` → `mode=today`，目標日期 = 今日，GMT+8 時間範圍 = 06:00–22:00
- 參數為 `tomorrow` → `mode=tomorrow`，目標日期 = 明日（今日 +1 天），GMT+8 時間範圍 = 04:00–22:00

**步驟一：並行抓取兩個來源**

ForexFactory：
1. 用 WebFetch 抓取 `https://nfs.faireconomy.media/ff_calendar_thisweek.json`
2. 篩選 `date` == **目標日期**（格式：`YYYY-MM-DD`）
3. 套用篩選邏輯 A、A2、A-USD、B
4. 時區轉換（夏令 ET+12h，冬令 ET+13h）→ 排序
5. **Tentative 事件識別（必做）：** ForexFactory JSON 不含 Tentative 標記，對 Tentative 事件仍給佔位時間戳，無法從 JSON 直接判斷。必須用 **Playwright 開啟 ForexFactory 網站**，對照通過篩選的事件，找出時間欄顯示為 "Tentative" 的項目，輸出時標示 `Tent.` 而非具體時間。若 Playwright 無法存取，在輸出中對 ForexFactory 來源的事件附註「⚠️ Tentative 狀態未驗證，請至 ForexFactory 確認」。

Myfxbook：
1. 用 Playwright 導航至 `https://www.myfxbook.com/forex-economic-calendar`
2. 等待 `tr.economicCalendarRow` 出現（頁面為動態 JS 載入，需等約 5–8 秒）
3. 用 JavaScript 提取事件，**欄位對應（新版頁面設計）：**
   ```javascript
   const rows = document.querySelectorAll('tr.economicCalendarRow');
   // tds[0] = 日期時間（格式 "Jun 11, 07:00"，已為 GMT+8）
   // tds[3] = 貨幣（"KRW"）
   // tds[4] = 事件名稱
   // tds[5] = 影響力（"HIGH"/"MEDIUM"/"LOW"/"NONE"）
   ```
4. **日期篩選（必做）：** `tds[0].innerText` 格式為 "Jun 11, 07:00"，先驗證日期部分（"Jun 11"）與目標日期吻合，再取 HH:MM，不符的事件一律丟棄
5. 套用相同篩選邏輯 A、A2、A-USD、B（時間已為 GMT+8，無需轉換）

> 若 KRW 未出現，依上方「KRW 缺失時的修復方法」做一次性 Filter 設定。

**步驟二：合併去重**

相同時間 ＋ 相同貨幣 ＋ 相似事件名稱 → 視為同一筆，保留一條。

**步驟三：時間範圍過濾（僅 `today` 與 `tomorrow` 模式）**
- 時區轉換完成後，同時檢查兩個條件，任一不符即丟棄：
  1. **日期必須等於目標日期**（GMT+8 換算後日期 ≠ 目標日期的事件一律排除，即使 clock time 在範圍內）
  2. **時間必須在範圍內**：
     - `today`：保留 06:00 ≤ 時間 ≤ 22:00
     - `tomorrow`：保留 04:00 ≤ 時間 ≤ 22:00
- `mode=default` 不做此過濾，顯示全天事件；跨日事件以 `(+1)` 標示

**步驟四：套用輸出格式規則後輸出表格**

## 時區說明

- **ForexFactory** 時間為 **ET（美東時間）**，輸出前轉換為 GMT+8：
  - 夏令時間（3月第二個週日 ～ 11月第一個週日）：ET = UTC-4，GMT+8 = ET + 12 小時
  - 冬令時間（其餘時間）：ET = UTC-5，GMT+8 = ET + 13 小時
- **Myfxbook** 時間已為 **GMT+8**（Playwright 瀏覽器依 cookie timezone=8.0 顯示），無需轉換
- 在輸出開頭標示：`時區：GMT+8（台北）`

## 輸出範本

標題依模式變化：
- `/news` → `📅 今日 XAUUSD 高影響力事件（YYYY-MM-DD）`
- `/news today` → `📅 今日 XAUUSD 高影響力事件（YYYY-MM-DD，06:00–22:00 GMT+8）`
- `/news tomorrow` → `📅 明日 XAUUSD 高影響力事件（YYYY-MM-DD，04:00–22:00 GMT+8）`

```
📅 今日 XAUUSD 高影響力事件（YYYY-MM-DD）
時區：GMT+8（台北）

| 時間（GMT+8） | 事件                                      | 影響力 |
|---|---|---|
| All Day        | USD-Some Holiday Event                    | High   |
| Tent.(21:00)   | USD-Cleveland Fed Inflation Expectations  | Low    |
| Tent.(找不到)  | USD-Some Other Tentative Event            | Low    |
| 14:00   | EUR-Balance of Trade                      | High   |
| 14:30   | HUF-Inflation Rate YoY (Apr)              | Low    |
|         | HUF-Core Inflation Rate YoY (Apr)         | Low    |
| 15:05   | ***EUR-ECB de Guindos Speech              | High   |
| 17:00   | AOA-Inflation Rate MoM (Apr)              | Low    |
|         | EUR-Inflation Rate MoM (Apr)              | Low    |
|         | EUR-Inflation Rate YoY (Apr)              | Low    |
| 17:45   | ***USD-FOMC Member Cook Speaks            | Low    |
| 20:30   | CAD-Employment Change (Apr)               | High   |
|         | CAD-Unemployment Rate (Apr)               | High   |
|         | USD-Non-Farm Employment Change (Apr)      | High   |
|         | USD-Unemployment Rate (Apr)               | High   |
| 07:30 (+1) | ***USD-FOMC Member Bowman Speaks       | Low    |
|            | ***USD-FOMC Member Daly Speaks         | Low    |
```

若今日無符合事件，回覆：「今日無高影響力事件。」
