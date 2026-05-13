---
name: news
description: 抓取外匯經濟日曆，篩選對 XAUUSD（黃金）高影響力的事件並依時間列出。使用 /news 觸發（預設今日全天）、/news today（今日 04:00–22:00 GMT+8）、/news tomorrow（明日 04:00–22:00 GMT+8）。當用戶輸入 /news 系列指令時立即執行，無需額外說明或確認。資料來源為 ForexFactory JSON API 與 Myfxbook，自動套用 XAUUSD 篩選邏輯。
---

# /news — XAUUSD 高影響力經濟日曆

## 呼叫模式與時間範圍

| 指令 | 目標日期 | GMT+8 時間範圍 |
|------|----------|----------------|
| `/news` | 今日 | 全天（無限制） |
| `/news today` | 今日 | 04:00 – 22:00 |
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
- **Myfxbook filter 解法（必做）**：頁面初次載入後預設只顯示約 15 個主要國家事件（約 62 筆）。必須在頁面載入後執行 `location.reload(true)` 強制重載，讓 `loggedOffCalendar` cookie（含所有貨幣清單）生效，重載後事件數會增至 120+ 筆，HUF、SCR 等非主要貨幣事件才會出現
- Myfxbook 時間已為 **GMT+8**，無需額外轉換（已驗證：CAD Employment 20:30、ECB Lagarde 15:00 均與 ForexFactory 換算結果吻合）

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

### B. 所有貨幣 — 高影響力全納入
**所有貨幣**（不限五大貨幣）中，`impact == "High"` 的事件，不論名稱為何，一律納入。

> ⚠️ 注意：舊版規則 B 只限 AUD/CAD/EUR/GBP/USD，已修正為**全貨幣適用**。

## 輸出格式規則

### 1. 同一時間只顯示一條代表事件
相同時間有多個符合篩選的事件，**只保留一條代表即可，其餘省略**（任意選擇）。

**例外：演講類事件（title 含 speech / speaks）不受此限，所有符合 A2 規則的演講都必須完整列出。**
若同一時間同時有普通事件與演講，演講全部列出，普通事件只留一條代表。

**正確示範（同時間只顯示代表）：**
```
| 17:00 | EUR-Employment Change QoQ (Q1)      | High |
| 20:30 | USD-PPI MoM (Apr)                   | High |
```

**錯誤示範（同時間列出所有事件）：**
```
| 17:00 | EUR-GDP Growth Rate QoQ (Q1)        | Low  |
|       | EUR-GDP Growth Rate YoY (Q1)        | Low  |
|       | EUR-Employment Change QoQ (Q1)      | High |
|       | EUR-Employment Change YoY (Q1)      | High |
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
- 時間為 Tentative → 標示 `Tent.`
- Holiday 全天事件 → 若名稱不含篩選關鍵字且非 High impact，略過不顯示
- 換算後超過 24:00（隔日台北時間）→ 標示 `(+1)`
- 預期值與前值**不顯示**
- 表格後**不加**今日重點摘要或任何總結

## 執行步驟

**步驟 0：判斷模式**
- 無參數 → `mode=default`，目標日期 = 今日，不套用時間範圍過濾
- 參數為 `today` → `mode=today`，目標日期 = 今日，GMT+8 時間範圍 = 04:00–22:00
- 參數為 `tomorrow` → `mode=tomorrow`，目標日期 = 明日（今日 +1 天），GMT+8 時間範圍 = 04:00–22:00

**步驟一：並行抓取兩個來源**

ForexFactory：
1. 用 WebFetch 抓取 `https://nfs.faireconomy.media/ff_calendar_thisweek.json`
2. 篩選 `date` == **目標日期**（格式：`YYYY-MM-DD`）
3. 套用篩選邏輯 A、A2、B
4. 時區轉換（夏令 ET+12h，冬令 ET+13h）→ 排序
5. **Tentative 事件識別（必做）：** ForexFactory JSON 不含 Tentative 標記，對 Tentative 事件仍給佔位時間戳，無法從 JSON 直接判斷。必須用 **Playwright 開啟 ForexFactory 網站**，對照通過篩選的事件，找出時間欄顯示為 "Tentative" 的項目，輸出時標示 `Tent.` 而非具體時間。若 Playwright 無法存取，在輸出中對 ForexFactory 來源的事件附註「⚠️ Tentative 狀態未驗證，請至 ForexFactory 確認」。

Myfxbook：
1. 用 Playwright 導航至 `https://www.myfxbook.com/forex-economic-calendar`
2. **執行 `location.reload(true)` 強制重載**，等待頁面完成載入
3. 用 JavaScript 提取所有含**目標日期**的事件文字（`tomorrow` 模式需識別明日日期區塊）
4. 驗證事件數量是否達 100+ 筆（若仍只有 60 餘筆，表示 reload 未生效，需重試）
5. 套用相同篩選邏輯 A、A2、B（時間已為 GMT+8，無需轉換）

> ⚠️ **Myfxbook 日期驗證（必做，防止跨日誤抓）：**
> Myfxbook 以 UTC 日期決定事件歸屬的 date header，但顯示時間已轉 GMT+8。UTC 日期是目標日（如 May 11）但 GMT+8 顯示為隔日（如 May 12, 05:00）的事件，會被塞在目標日的 header 區段下，卻屬於隔日事件。
>
> **提取規則：** 從 `calendarDateTd` 取完整 innerText（格式 "May 11, 09:30"），**先驗證日期部分（"May 11"）與目標日期吻合**，再取 HH:MM 部分。顯示日期不符的事件一律丟棄，不論它在哪個 date header 區段下。
>
> **範例錯誤（需丟棄）：** `dateAttr="2026-05-11 21:00"` 但 `innerText="May 12, 05:00"` → 顯示是 May 12，丟棄。

**步驟二：合併去重 + 同時間取代表**

- 相同時間 ＋ 相同貨幣 ＋ 相似事件名稱 → 視為同一筆，保留一條。
- 同一時間有多筆不同事件通過篩選 → 只保留一條代表（任意選），其餘省略。
- 例外：演講類（speech / speaks）符合 A2 規則的全部保留，不受「只留一條」限制。

**步驟三：時間範圍過濾（僅 `today` 與 `tomorrow` 模式）**
- 時區轉換完成後，移除 GMT+8 時間不在範圍內的事件：
  - `today`：保留 04:00 ≤ 時間 ≤ 22:00
  - `tomorrow`：保留 04:00 ≤ 時間 ≤ 22:00
- `mode=default` 不做此過濾，顯示全天事件

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
- `/news today` → `📅 今日 XAUUSD 高影響力事件（YYYY-MM-DD，04:00–22:00 GMT+8）`
- `/news tomorrow` → `📅 明日 XAUUSD 高影響力事件（YYYY-MM-DD，04:00–22:00 GMT+8）`

```
📅 今日 XAUUSD 高影響力事件（YYYY-MM-DD）
時區：GMT+8（台北）

| 時間（GMT+8） | 事件                                      | 影響力 |
|---|---|---|
| All Day | USD-Some Holiday Event                    | High   |
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
