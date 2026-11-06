# 助手對話入口（引導式一問一答）＋ 導覽調整 — 設計規格

- 日期：2026-10-05
- 狀態：設計已核可，待規格複核
- 分支基準：`master`
- 目標分支：`feat/assistant-flow`
- 備註：本規格為第一份；「我的回報」完整版另立第二份（見 §8）。

---

## 1. 背景與目的

首頁三張任務卡（回報社區問題／附近事件／福利活動）目前只是**導覽**：點下去進到各頁，使用者還得自己想「要輸入什麼」。同時桌面頂部導覽重複了這三個功能，而「我的回報」「事件總覽」這類入口卻不存在。

**目的**：
1. 點卡片即開始一段**以該任務為題的引導式對話**（一問一答），問完**接手既有流程**。
2. 調整導覽：桌面頂部移除重複的三項，改放「事件總覽」與「我的回報」；手機底部三項改為觸發對話。

---

## 2. 範圍

### 2.1 納入（In scope）
- 新增對話頁路由 `/assistant/:task`（task ∈ `report` | `events` | `welfare`）。
- 新增 `AssistantView.vue`：依「任務腳本」渲染一問一答。
- 新增任務腳本資料 `services/assistantScripts.js`（步驟與接手動作以資料描述）。
- 首頁三張 `BigTaskCard` 改連到對話頁（`/assistant/report` 等）。
- 問完的答案寫入對應 store，並導向既有流程／結果。
- **桌面 `TopNav`**：移除「回報問題／附近事件／福利活動」三個項目；改為顯示品牌 + 「事件總覽」(→ `/events`) + 「我的回報」(→ `/my-reports`)。
- **手機 `BottomNav`**：三個功能項（回報／事件／福利）改為觸發對應對話（連到 `/assistant/:task`），保留「首頁」。
- **「我的回報」最小版**：新增 `/my-reports` 頁，顯示**本工作階段**送出的回報（空狀態時顯示提示），供 header 入口使用。

### 2.2 排除（Out of scope，後續另案）
- 真 LLM／多輪自由聊天。
- 修改既有流程內部（回報 3 步、事件清單、福利搜尋的行為不變）。
- 語音、圖片／音訊在對話中的新用法（回報的描述步驟沿用既有 `AiPromptBar`，含其既有附件能力）。
- 帳號、記憶跨工作階段的對話。
- **首頁萬用輸入列（`AiPromptBar`）的複合需求處理**（多意圖分流與結果整合）；本次只做卡片的單一業務對話，輸入列維持現狀。
- **「我的回報」完整版**（跨重新整理的本機紀錄、狀態更新、更完整清單）→ 第二份規格。

---

## 3. 現況

- `HomeView.vue` 三張 `BigTaskCard` → `/report`、`/events`、`/welfare`。
- `TopNav.vue` 顯示品牌 + `NAV_ITEMS` 三項（與首頁卡片重複）；`BottomNav.vue` 顯示 `MOBILE_NAV_ITEMS`（首頁 + 三項）。
- `ClarifyView.vue`：**只在意圖不明時**問兩層。定位與「點卡片主動開始」不同。
- `intentService`／`intentRules`：由文字分類意圖（規則驅動）。
- 回報送出：`report/submit` → `eventService.report(draft)` → `eventStore.add`，記下 `submittedId`；`DoneView` 回首頁時 `report/reset` 清空。
- 交接資料面：
  - 回報：`report/setDescription` → `/report/location`（路由護欄要求 description 非空）。
  - 事件：`query/setRegion`、`query/setFilters({time|type|status})` → `/events`。
  - 福利：`welfare/search(keyword)`；`WelfareView` 支援 `?q=` 自動搜尋。

---

## 4. 目標設計

### 4.1 互動（引導式一問一答）
- 一次只問**一個**問題；選項題給**大按鈕**，必要時可打字。
- 顯示進度「第 N 題 / 共 M 題」。
- 每步都有「回首頁」。
- 引導式，非自由聊天：避免使用者亂打後卡住。
- 用語白話（例如「我來幫你回報問題…」）；**不出現 Agent／AI／API 等技術詞**（G5）。

### 4.2 路由與元件
- 路由：`/assistant/:task`（`task` ∈ `report` | `events` | `welfare`）；無效的 `task` 導回首頁。
- `AssistantView.vue`
  - 讀取該任務腳本；維護 `stepIndex` 與 `answers`（元件本地狀態）。
  - 依步驟型別渲染：`text`（沿用既有的 `AiPromptBar`，保留文字／語音／附件能力）或 `choice`（大按鈕選項，可含「都可以」）。
  - 完成所有步驟後，套用腳本 `finish(answers)` 回傳的 **commits**（寫入 store），再 `router.push` 其 **route**。
  - 「回首頁」導向 `/`。
- 狀態：無效 task → 回首頁；答案保留於本次對話（回首頁即結束）。

### 4.3 任務腳本（`services/assistantScripts.js`，純資料 + 純函式）
每個任務為 `{ title, intro, steps, finish }`：

| task | intro | steps | finish（commits / route） |
|---|---|---|---|
| `report` | 「我來幫你回報問題。先說說看到什麼狀況，一句話就好。」 | 1. `text`「看到什麼狀況？」（必填，placeholder 同既有） | commit `report/setDescription` → route `report-location` |
| `events` | 「我來幫你找附近的事件。先問你兩個問題。」 | 1. `choice`「想找多久以內？」（今天／這週／這個月／全部）<br>2. `choice`「哪一區？」（中華路／汀州路／青年公園／台北車站／都可以） | commits `query/setFilters({time})`（+ `query/setRegion` 若非空）→ route `events` |
| `welfare` | 「我來幫你找福利或活動。」 | 1. `choice`「想找哪一類？」（補助／課程／活動／健康檢查／都可以）<br>2. `choice`「哪一區？」（同上／都可以） | route `welfare`（`?q=` = 非空答案以空白連接） |

- `finish(answers)` 為**純函式**，回傳 `{ commits: Array<{type,payload}>, route: RouteLocation }`，不直接碰 store／router（由元件套用），方便測試。
- **回報的描述步驟**沿用 `AiPromptBar`：其附件（拍照／語音）由元件以與 `DescribeStep` 相同的方式寫入 `report` store（`setPhoto`／`setAudio`），因此「接手既有流程」不會失去附件能力。
- 步驟與選項資料需與既有 store／服務用語一致（例如地區用語須與事件／福利種子資料一致）。

### 4.4 對既有流程的影響
- **不改**既有流程行為；只把答案寫進 store 再導頁。
- 回報：因路由護欄要求 description 非空，對話必定先收集描述才導向 `report-location`。
- 事件：先寫入 store 再 `push`，`EventListView.created()` 會以當前篩選載入。
- 福利：以 `?q=` 導向，`WelfareView.created()` 自動搜尋（沿用既有機制）。

### 4.5 無障礙
- 每個選項／按鈕 ≥ 48px、對比 AA、可見焦點。
- 步驟切換後把焦點移到新問題的標題或輸入（避免焦點遺失）。
- 尊重 `prefers-reduced-motion`；沿用入口基準 token。
- 語意：問題以 `h1`／`h2` 標示；進度以文字（非僅視覺）。

### 4.6 定位區隔：快速服務 vs 萬用輸入列
- **三張卡（快速服務）**：只處理**個別業務能力**（回報／事件／福利），走引導式對話 → 接手既有流程。
- **底部 AI 輸入列（萬用輸入）**：自由描述，定位為可承載**複合需求**（一句話同時提到多件事）。
  - **現況**：只做**單一意圖分流**（申報／查詢／福利），無法判斷時進 `ClarifyView` 釐清；尚未支援複合需求。
  - **本次不變更輸入列**；複合需求列為後續（§8）。

### 4.7 導覽調整
- **桌面 `TopNav`**：移除「回報問題／附近事件／福利活動」三個項目；改為顯示品牌（logo + 名稱，點擊回首頁）+「事件總覽」(→ `/events`) +「我的回報」(→ `/my-reports`)。
- **手機 `BottomNav`**：保留「首頁」；「回報問題／附近事件／福利活動」三項改為連到 `/assistant/:task`（點擊即開始該任務對話）。
- 既有頁面路由（`/report`、`/events`、`/welfare`）不變，仍為對話的接手目標與直接連結（例如空狀態的「回報一個問題」）。

### 4.8 我的回報（最小版，本工作階段）
- 新增 `/my-reports` 頁，列出**本工作階段**送出的回報（標題／類型、地點、時間、狀態）。
- 資料來源：新增 `myReports` store 模組（**記憶體**）；於回報送出成功時加入一筆快照（`{ id, title, type, placeText, timeText, status, reportedAt }`）。
- 沒有紀錄時顯示 `EmptyState`（「還沒有回報紀錄」＋ 前往回報的按鈕）。
- **跨重新整理的本機紀錄與更完整清單**屬第二份規格（§8）；本版不做持久化。

---

## 5. 技術決策

1. **規則驅動**：純前端腳本模擬對話，非真 LLM；維持 `services/` 為未來接真後端的接縫。
2. **腳本以資料描述 + 純 `finish`**：新增任務或改問題只改資料；邏輯可單元測試。
3. **不重用 `ClarifyView`**：其語意為「意圖不明才問」，與「主動開始對話」不同；本功能另立元件，避免一元件背兩種職責。
4. **「我的回報」最小版用記憶體 store**：先對齊「導覽入口可用」；持久化與狀態更新留給第二份規格，避免本份範圍膨脹。

---

## 6. 影響檔案

- Create：`app/src/views/assistant/AssistantView.vue`
- Create：`app/src/services/assistantScripts.js`
- Create：`app/src/views/my/MyReportsView.vue`
- Create：`app/src/store/modules/myReports.js`
- Modify：`app/src/router/index.js`（新增 `/assistant/:task`、`/my-reports`；`MOBILE_NAV_ITEMS` 的 to 改為對話頁）
- Modify：`app/src/store/index.js`（註冊 `myReports` 模組）
- Modify：`app/src/store/modules/report.js`（送出成功時加入 `myReports`）
- Modify：`app/src/views/home/HomeView.vue`（卡片改連對話頁）
- Modify：`app/src/components/TopNav.vue`（移除三個項目，改放事件總覽／我的回報）
- Modify：`app/src/components/BottomNav.vue`（三項改觸發對話）
- Create／Modify：對應測試（`assistantScripts.spec.js`、`assistantView.spec.js`、`myReports.spec.js`、`homeView.spec.js`、`nav.spec.js`）

---

## 7. 驗收標準

- [ ] 點首頁任一張卡 → 進入該任務的對話頁（不是直接進既有頁）。
- [ ] 一問一答：一次顯示一個問題；選項題為大按鈕、文字題可打字。
- [ ] 完成後接手正確流程：回報→選位置；事件→已套篩選的清單；福利→已搜尋的結果。
- [ ] 無效的 `:task` 導回首頁，不報錯。
- [ ] 桌面頂部導覽顯示「事件總覽」(→ `/events`) 與「我的回報」(→ `/my-reports`)；不再顯示原本三個功能項。
- [ ] 手機底部導覽「回報／事件／福利」三項點擊後開始對應對話；「首頁」仍可回首頁。
- [ ] 送出回報後，`/my-reports` 會列出該筆（本工作階段）；沒有紀錄時顯示空狀態。
- [ ] 畫面不出現 Agent／AI／API 等技術詞（G5）。
- [ ] 鍵盤可完成整個對話；焦點在步驟切換後不遺失。
- [ ] 既有測試維持通過；`npm run build` 成功。

---

## 8. 後續（非本次）

- 在對話中加入語音輸入、進度記憶、自由文字混合。
- 事件／福利對話的第 2 題之後視需要增減。
- 真後端／LLM 接縫。
- **首頁萬用輸入列的複合需求**：多意圖分流與結果整合。
- **第二份規格：「我的回報」完整版** — 跨重新整理的本機紀錄（例如 localStorage）、狀態更新、更完整清單／篩選。

---

## 9. 參考

- 視覺 mock：`.superpowers/brainstorm/assistant-flow/content/assistant-flow.html`
- 現有流程：`app/src/views/report/DescribeStep.vue`、`app/src/views/query/EventListView.vue`、`app/src/views/welfare/WelfareView.vue`
- 意圖與釐清：`app/src/services/intentRules.js`、`app/src/views/clarify/ClarifyView.vue`
- 產品定位：`Session1_產品目標與範圍.md` §7、G5（畫面不得出現技術詞）
