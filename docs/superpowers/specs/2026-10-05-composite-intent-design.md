# 複合需求入口（一句話多任務）＋移除釐清 — 設計規格

- 日期：2026-10-05
- 狀態：設計已核可，待規格複核
- 分支基準：`master`
- 目標分支：`feat/composite-intent`

---

## 1. 背景與目的

首頁萬用輸入列目前只能處理**單一意圖**；判斷不出來時會跳到釐清畫面（「你是想找哪一種」）。

**目的**：
1. 讓輸入列能處理**一句話包含多個任務**（複合需求），一次給出綜合結果。
2. **移除釐清畫面**（`/clarify` 與「你是想找哪一種」），改由綜合結果頁承接。

---

## 2. 範圍

### 2.1 納入（In scope）
- 新增**多意圖解析**：把輸入切成子句、各別分類、彙整不重複的意圖。
- 首頁輸入列送出後的分流：
  - **0 個意圖** → `/results`（顯示提示＋三個任務入口）。
  - **1 個意圖** → 直接進對應流程（沿用現行：回報→回報流程、事件→事件清單、福利→福利搜尋）。
  - **2 個以上** → `/results` 綜合結果頁。
- 新增 `/results` 綜合結果頁：分區呈現事件／福利結果，並為回報意圖顯示入口卡。
- 移除釐清：`ClarifyView`、`/clarify` 路由、`intent/resolveChoice` 與其測試。

### 2.2 排除（Out of scope）
- 真 LLM／語意模型（維持規則驅動）。
- 三個卡片（助手對話）不變——複合需求只走輸入列。
- 跨任務的結果排序／合併成單一清單（本次為分區呈現）。
- 「我的回報」完整版（另一份規格）。

---

## 3. 現況

- `services/intentRules.js`：`classifyText(text)` 以規則回傳 `{ intent, confidence, extracted }`（welfare／query／report／ambiguous）。
- `store/modules/intent.js`：`routeFromText` 把 `intent` 映射到路由（ambiguous → `clarify`）；`resolveChoice` 處理釐清畫面的兩層選擇。
- `views/clarify/ClarifyView.vue`：釐清畫面。
- `HomeView.submit`：`dispatch('intent/routeFromText', text)` 後 `$router.push(route)`。
- 事件查詢：`query/searchFromText(text)`；福利搜尋：`welfare/search(keyword)`。

---

## 4. 目標設計

### 4.1 多意圖解析（`intentRules.js`）
- 新增純函式 `parseIntents(text) -> Array<{ intent, text }>`：
  1. **切子句**：依標點（`，、,。；;`、換行）與連接詞（`順便`、`還有`、`以及`、`另外`、`也要`、`然後`）切分。
  2. 對每個子句跑 `classifyText`；略過 `ambiguous` 的子句。
  3. 依出現順序收集**不重複的意圖**（同意圖只留第一個子句文字）。
- 保留既有 `classifyText` 行為（供解析使用）。

### 4.2 分流（`store/modules/intent.js` 的 `routeFromText`）
```
intents = parseIntents(text)
0 個  → { name: 'results', query: { q: text } }
1 個  → report  → commit report/setDescription(clause) → { name: 'report' }
        query   → { name: 'events',  query: { q: clause } }
        welfare → { name: 'welfare', query: { q: clause } }
2+ 個 → { name: 'results', query: { q: text } }
```
- 移除 `ROUTE_BY_INTENT` 的 `ambiguous: 'clarify'` 與 `resolveChoice`。
- `remember` 改記錄 `{ text, intents }`（`intents` 為解析後清單）。

### 4.3 綜合結果頁（`/results`）
- 讀 `$route.query.q` → `parseIntents`。
- 分區呈現：
  - **事件區**（有 query 意圖）：以該子句 `dispatch('query/searchFromText', clause)`；呈現載入／錯誤／空狀態／`ResultCard` 清單。
  - **福利區**（有 welfare 意圖）：以該子句 `dispatch('welfare/search', clause)`；呈現 `SearchingState`／`NoResultState`／`ResultList`。
  - **回報區**（有 report 意圖）：一張入口卡「要回報嗎？前往回報」→ `/assistant/report`（不自動開始流程）。
  - **0 意圖**：溫和提示「看不太懂，換句話說或從下面選」＋三個任務入口（回報／事件／福利）。
- 各區塊標題用白話（「附近事件」「福利與活動」），不出現技術詞（G5）。

### 4.4 移除釐清
- 刪 `views/clarify/ClarifyView.vue` 與其測試。
- `router/index.js` 移除 `/clarify` 路由與 import。
- `intent.spec.js` 移除 `resolveChoice` 測試並更新 `routeFromText` 測試（ambiguous → `results`）。

### 4.5 無障礙
- 各區塊用正確標題層級（`h1` 頁標、`h2` 區塊）；載入用 `role="status"`。
- 沿用入口基準 token；可點元素 ≥48px、對比 AA、可見焦點、`prefers-reduced-motion`。

---

## 5. 技術決策

1. **規則驅動**：多意圖以標點／連接詞切分 + 既有 `classifyText`，不做真語意模型。
2. **單一意圖不走綜合頁**：維持既有直達體驗；只有複合或無法判斷才進 `/results`。
3. **以 `?q=` 攜帶原話**：`/results` 可深連結、可重整（重用既有 `?q=` 慣例）。
4. **重用既有服務與元件**：事件用 `query/searchFromText`、福利用 `welfare/search`；結果用既有 `ResultCard`／`ResultList`／`NoResultState`／`SearchingState`／`ErrorAlert`。

---

## 6. 影響檔案

- Modify：`app/src/services/intentRules.js`（新增 `parseIntents`）
- Create：`app/src/services/__tests__/intentRules.spec.js`
- Modify：`app/src/store/modules/intent.js`（`routeFromText` 改多意圖；移除 `resolveChoice`）
- Modify：`app/src/store/__tests__/intent.spec.js`
- Create：`app/src/views/results/ResultsView.vue`
- Create：`app/src/views/results/__tests__/resultsView.spec.js`
- Modify：`app/src/router/index.js`（新增 `/results`、移除 `/clarify`）
- Delete：`app/src/views/clarify/ClarifyView.vue`、`app/src/views/clarify/__tests__/clarifyView.spec.js`

---

## 7. 驗收標準

- [ ] 一句話含多個任務（例如「附近有沒有積水，這個月有什麼老人活動」）→ 進 `/results`，事件與福利**分區**列出。
- [ ] 單一任務 → 直接進對應流程（不經 `/results`）。
- [ ] 無法判斷 → 進 `/results`，顯示提示與三個任務入口，不再出現「你是想找哪一種」。
- [ ] 句中含回報 → 綜合頁顯示「要回報嗎？前往回報」入口，不自動開始流程。
- [ ] `/clarify` 路由與畫面已移除，全站不再有此入口。
- [ ] 畫面不出現 Agent／AI／API 等技術詞（G5）。
- [ ] 既有測試維持通過；`npm run build` 成功。

---

## 8. 後續（非本次）

- 綜合結果的跨任務合併／排序；真語意模型；結果頁的篩選與再搜尋。
- 「我的回報」完整版。

---

## 9. 參考

- 現況：`app/src/services/intentRules.js`、`app/src/store/modules/intent.js`、`app/src/views/clarify/ClarifyView.vue`
- 資料來源：`query/searchFromText`、`welfare/search`
- 產品定位：`Session3` §D1（任務優先、自然語言為輔）
