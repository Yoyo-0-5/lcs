# Session 7：系統架構（Software Architecture）

> 本文件將 S1–S6 的規劃，落地為一份**可實作的技術架構**：純前端 SPA ＋ mock 資料層，並以可替換介面包裝所有「智慧」與外部服務。
>
> 依據：`Session1_產品目標與範圍.md`、`Session2_使用情境與互動需求.md`、`Session3_介面規劃.md`、`Session4_UX規劃.md`、`Session5_設計方向_Firefox風格.md`、`Session6_使用者流程設計.md`。
>
> 產出時機：本文件經確認後，才由 `writing-plans` 產出實作計畫。

---

## 0. 範圍、形態與全域限制

### 0.1 落地形態

**課程／展示用 MVP**：重點是流程端到端可跑通、可 demo；低部署門檻（靜態檔案即可）。

### 0.2 本次做與不做

| 做（真實） | 不做（模擬／範圍外） |
|---|---|
| 三個入口、P0–P12 畫面與路由 | 後端服務、資料庫 |
| 申報／查詢／福利搜尋三支流程 | 真 LLM 呼叫 |
| 純匿名、表單驗證、loading／錯誤／空狀態 | 真的上網搜尋與多來源檢索 |
| 瀏覽器定位、語音輸入、Google Maps 顯示 | **真 multi-agent**（見附錄 A） |
| axios + mock adapter 的資料層 | 帳號／權限 |

### 0.3 全域限制（Global Constraints）

- **G1**：**每個檔案 < 200 行**；採**模組化**（單一職責、小檔案、明確邊界）。
- **G2**：元件**不得直接呼叫 axios**，一律經 store／service。
- **G3**：所有「智慧／外部」能力以**可替換介面包裝**（單一 `USE_MOCK` 開關），之後接真後端不需改 views／store。
- **G4**：延續 S4／S5 無障礙底線（對比 ≥4.5:1、focus 可見、觸控 ≥48px、`prefers-reduced-motion`）；品牌橘 `#FF7139` 僅裝飾。
- **G5**：畫面上**不得出現** Agent／API／Database／多代理等技術詞（S2 §24）。
- **G6**：**mobile-first**；需同時支援**手機與桌機兩種版型**（斷點 375／768／1024／1440，依 S3 §9）。
  - 手機（<768）：**底部浮動導覽**、單欄。
  - 桌機（1024+）：**頂部導覽（3 項）**，內容區加寬，適合處可用**雙欄**（卡片／結果）。

---

## 1. 系統總覽、技術棧與專案結構

### 1.1 分層原則

```text
Vue 元件 (views / components)
        │  只呼叫 store / service，不直接碰資料
        ▼
store/ (Vuex)  ── 流程狀態與 action
        ▼
services/  ── 業務邏輯 + 對外介面（可替換）
   intentService / eventService / welfareService / locationService / speechService
        │  axios 呼叫 /api/*
        ▼
api/http.js (axios instance)  ←  api/mock.js (mock adapter 攔截，模擬延遲/錯誤/逾時)
        ▼
mocks/*.json  (events / welfare / places / taxonomy)
```

### 1.2 技術棧（定案）

| 層 | 選擇 |
|---|---|
| 框架 | Vue **2.7**（Vue2 最終版，Vite 支援佳） |
| 建置 | Vite ＋ `@vitejs/plugin-vue2` |
| 套件管理 | pnpm |
| 元件庫 | Element UI 2.15.x（Vue2），SCSS 客製主題 |
| 路由 | Vue Router 3 |
| 狀態 | Vuex 3 |
| HTTP | axios ＋ axios-mock-adapter |
| 地圖 | Google Maps JS API（需金鑰） |
| 語音 | Web Speech API（`zh-TW`） |
| 資料 | 本地 JSON（mock） |
| 樣式 | SCSS ＋ CSS 變數（不用 Tailwind） |
| 測試 | Vitest（＋ `@vue/test-utils` 1.x，Vue2 相容） |

> 風險標記：Vue 2 官方已於 2023/12/31 EOL，不再有安全更新。本案沿用係依指定技術棧。

### 1.3 專案結構

```text
LCS_project/
├── Session1..7_*.md            ← 既有文件（不動）
├── design-system/              ← 既有 tokens（不動）
└── app/                        ← 新增：Vue 應用（與文件分離）
    ├── package.json / vite.config.js / index.html
    ├── .env.local              ← VITE_GOOGLE_MAPS_KEY
    └── src/
        ├── main.js  App.vue
        ├── router/index.js
        ├── store/
        │   ├── index.js
        │   └── modules/ (report.js, query.js, welfare.js, location.js, ui.js)
        ├── api/ (http.js, mock.js)
        ├── services/
        │   ├── intentService.js
        │   ├── eventService.js
        │   ├── welfareService.js
        │   ├── locationService.js
        │   └── speechService.js
        ├── mocks/ (events.json, welfare.json, places.json, taxonomy.json[, 依類別分檔])
        ├── views/
        │   ├── home/HomeView.vue
        │   ├── report/ (DescribeStep.vue, LocationStep.vue, ConfirmStep.vue, DoneView.vue)
        │   ├── query/ (EventListView.vue, EventDetailView.vue)
        │   ├── welfare/ (WelfareView.vue, SearchInput.vue, SearchingState.vue, ResultList.vue, NoResultState.vue)
        │   └── clarify/ClarifyView.vue
        ├── components/
        │   ├── AppShell.vue     ← 依斷點切換 TopNav／BottomNav
        │   ├── TopNav.vue       ← 桌機（1024+）頂部導覽
        │   ├── BottomNav.vue    ← 手機底部浮動導覽
        │   ├── BigTaskCard.vue  NlInputBox.vue  VoiceButton.vue
        │   ├── ResultCard.vue   SourceBadge.vue
        │   ├── StepIndicator.vue
        │   ├── EmptyState.vue   ErrorAlert.vue
        │   └── icons/ (House.vue, Flag.vue, MapPin.vue, Gift.vue, ArrowLeft.vue, Calendar.vue, Microphone.vue)
        ├── composables/ (useBreakpoint.js ← 響應式判斷)
        ├── styles/ (tokens.css ← 複製 S5、element-variables.scss、element-override.scss)
        └── utils/ (ranking.js, timeFilter.js, textExtract.js …)
```

檔案切分一律遵守 **G1（< 200 行）**；超大畫面依職責拆子元件，mock JSON 過大依類別分檔。

---

## 2. 路由、畫面與狀態

### 2.1 路由（Vue Router 3）

| 路徑 | 畫面 | 備註 |
|---|---|---|
| `/` | P0 首頁 Hub | 三卡片＋萬用輸入框 |
| `/report` | P1 描述事件 | 進入申報流程 |
| `/report/location` | P2 確認位置 | |
| `/report/confirm` | P3 摘要確認 | |
| `/report/done` | P4 完成 | 交接點提示 |
| `/events` | P5 查詢目前事件 | 篩選 chips |
| `/events/:id` | P7 事件詳情 | |
| `/welfare` | P8／P9／P10／P11 | 同一頁內依 `welfare.phase` 切換狀態 |
| `/clarify` | P12 釐清 | |

**流程護欄**：`/report/*` 以 `router.beforeEach` 檢查 Vuex `report.draft`；缺前一步資料則導回 `/report`（UX-6 保留狀態）。

### 2.2 Vuex 模組（各模組單檔 < 200 行）

| 模組 | state（要點） | 主要 actions |
|---|---|---|
| `report` | `draft{description, category, location, time, photo}`, `step`, `status` | `describe`, `resolveLocation`, `submit` |
| `query` | `region`, `filters`, `events`, `loading`, `error` | `loadEvents`, `setFilter`, `clearFilters` |
| `welfare` | `query`, `phase(idle/loading/processing/done)`, `results`, `timeout` | `search`, `retry`, `widen` |
| `location` | `current`, `candidates`, `selected`, `permission` | `requestCurrent`, `resolveText`, `pickCandidate` |
| `ui` | toast／錯誤 | `notify`, `clear` |

資料流一律：`views → dispatch action → services → api(axios+mock adapter)`（**G2**）。

### 2.3 響應式版型（mobile-first，見 G6）

| 斷點 | 導覽 | 版面 |
|---|---|---|
| 375px（基準，小手機） | 底部浮動導覽（3 項） | 單欄；三卡片直向堆疊 |
| 768px（平板） | 底部導覽 | 單欄加寬、內容置中 |
| 1024px+（桌機） | **頂部導覽（3 項）** | 內容加寬；適合處**雙欄**（如結果列表／卡片區） |
| 1440px | 頂部導覽 | 內容最大寬度收斂，避免過寬行長 |

**實作要點**
- 以 CSS media query ＋ 一個 `layout` 判斷（`store/ui` 或 composable）切換 `BottomNav`／`TopNav`；兩者共用同一組導覽項目。
- 元件一律**先寫手機版**，桌機為增強；不因桌機而移除手機可用性。
- 焦點可見與 `scroll-padding-bottom` 兩種版型都要成立（WCAG 2.4.11）。
- 觸控目標 ≥48px 在兩種版型一致。

---

## 3. 服務層、Mock 與「Agent 接縫」

### 3.1 Service 介面（＝未來的 Agent 接縫）

| Service | 介面（輸入 → 輸出） | MVP 實作 |
|---|---|---|
| `intentService` | `classify(text) → {intent: report｜query｜welfare｜ambiguous, confidence, extracted}` | mock：關鍵字／句型規則 |
| `eventService` | `report(draft) → event`；`list({region,time,type,status}) → event[]`；`get(id) → event` | mock：`events.json`＋延遲 |
| `welfareService` | `search({type,target,time,region}) → {results, partial, failedSources}` | mock：篩選 `welfare.json`＋去重／時效／排序 |
| `locationService` | `geocode(text) → candidate[]`；`reverse(lat,lng) → address` | 半真：Google Geocoding（真）＋`places.json` |
| `speechService` | `start()／stop()／onresult` | 真實：Web Speech API 包裝 |

**可替換設計（G3）**：每個 service 以單一 `USE_MOCK` 開關切換 mock／real；換真後端只改 service 實作，views／store 不動。

### 3.2 `api/`

- `http.js`：axios instance（`baseURL: '/api'`、timeout、攔截器）。
- `mock.js`：axios-mock-adapter 註冊 `/api/*`，回傳 `mocks/*.json`，可模擬**延遲、錯誤、空結果、逾時**。

### 3.3 Mock 資料

`src/mocks/`：`events.json`、`welfare.json`、`places.json`、`taxonomy.json`（必要時依類別分檔）。

### 3.4 Demo 情境開關

以 `?mock=empty / timeout / partial / error / ambiguous`（或 localStorage）**強制觸發**對應狀態，讓每一種狀態都能現場 demo（P9 逾時、P11 無結果、部分來源失敗、P12 釐清）。

### 3.5 對應 S6 的模擬點

| S6 流程 | 由誰模擬 |
|---|---|
| §1 意圖分流（ambiguous → P12） | `intentService.classify` |
| §2 事件理解／缺資訊、位置決策 | `eventService`、`locationService` |
| §4 三段式進度／逾時／部分失敗／無可靠結果 | `welfareService` ＋ demo 情境開關 |

---

## 4. 設計系統對應、圖示與測試

### 4.1 Design tokens → Element UI

- `src/styles/tokens.css`：沿用 S5 `tokens.css` 的 **Base＋Application 兩層 CSS 變數**（去 Tailwind `@theme` 段落）。
- `src/styles/element-variables.scss`：SCSS 覆寫 Element UI 主題 —— `$--color-primary: #a44900`、圓角、基本字級。
- `src/styles/element-override.scss`：對齊 tokens／高齡規則：
  - 按鈕 ≥52px、pill 圓角；輸入框 ≥48px、字級 ≥18px
  - `:focus-visible` 用 `--ring`（暖色）可見 focus
  - 手機浮動底部導覽 `fixed` ＋ `scroll-padding-bottom`；桌機頂部導覽 `sticky`（WCAG 2.4.11）
  - 內文 18px 起、行高 1.75、Noto Sans TC ＋ Atkinson Hyperlegible；mobile-first，桌機雙欄為增強

> Element UI 2.x 以 SCSS 變數客製（需 `sass`）；不使用 Tailwind。

### 4.2 圖示

S4 §10 要求統一線性；Element UI 內建圖示風格不一致、Phosphor 無官方 Vue2 版。故**自建內嵌 SVG 元件** `components/icons/`，統一 `stroke`、weight、24px（見 §1.3 清單）。

### 4.3 測試（Vitest，只測核心邏輯）

| 對象 | 測什麼 |
|---|---|
| `intentService.classify` | 分類正確、ambiguous → P12 |
| `locationService` | 候選決策、**S6 §14.4 低精度／未確認** |
| `welfareService` | 去重／時效／排序／無可靠結果 |
| `eventService` | 篩選、空狀態 |
| store actions | 對應流程狀態 |

UI 與版面以手動 demo 驗收（依 S4 §12 檢查清單）。

### 4.4 無障礙底線（沿用 S4／S5，見 G4）

對比 ≥4.5:1、focus 可見、觸控 ≥48px、`prefers-reduced-motion`；品牌橘 `#FF7139` 不承載文字／按鈕底。

---

## 5. 決策總表

| # | 項目 | 決策 |
|---|---|---|
| 1 | 落地形態 | 課程／展示用 MVP |
| 2 | 架構 | 純前端 SPA ＋ mock 資料層 |
| 3 | 框架／建置／套件 | Vue 2.7 ＋ Vite（`@vitejs/plugin-vue2`）＋ pnpm |
| 4 | 元件庫／樣式 | Element UI 2.15.x ＋ SCSS 客製；不用 Tailwind |
| 5 | 路由／狀態 | Vue Router 3 ＋ Vuex 3 |
| 6 | 資料層 | axios ＋ axios-mock-adapter ＋ 本地 JSON |
| 7 | 地圖／定位 | Google Maps JS API ＋ 瀏覽器 Geolocation |
| 8 | 語音 | Web Speech API（`zh-TW`） |
| 9 | 設計語言 | 沿用 S5 Firefox tokens（CSS 變數）＋ Element UI 主題 |
| 10 | 圖示 | 自建內嵌 SVG 線性圖示 |
| 11 | 測試 | Vitest 核心邏輯；UI 手動驗收 |
| 12 | multi-agent | **範圍外**（見附錄 A）；只留 service 接縫 |
| 13 | 檔案限制 | **每檔 < 200 行**、模組化（G1） |
| 14 | 應用位置 | `app/` 子目錄 |
| 15 | 版型 | **mobile-first**；手機底部導覽／桌機頂部導覽＋雙欄（G6） |

---

## 附錄 A：Multi-agent（範圍外，保留接縫）

依 S1 §7、S2 §23–24，AI／Agent 是部分功能的重要支撐；但本 MVP 為純前端，**不實作 multi-agent**。

- **本次**：Session6 流程中「系統理解需求」「搜尋整理」「地點解析」等系統職責，由 `services/` 的 **mock** 實作。
- **接縫**：未來真 multi-agent 直接接在 `services/` 介面之下；views／store 不需改動（G3）。
- **未來（獨立 spec）**：Agent 數量、職責分工、協作流程、框架、LLM 選型、真網頁檢索與來源驗證——待真有後端時另立文件。

```text
Session6 flow 圖的「系統」方塊
        ▲ 未來 multi-agent 就住在這裡
services/  ← 本次以 mock 實作（接縫）
```

## 附錄 B：與其他文件的對應與待決

| 本文件 | 來源 |
|---|---|
| §1 架構／技術棧 | S1 §7–8；S2 §27；S3 D9／D10；S5 §5、§7 |
| §2 路由／狀態 | S3 §1–3；S4 §4；S6 §1–4 |
| §3 服務／Mock | S6 §2.3、§3、§4；S2 §16、§19、§22 |
| §4 設計／測試 | S4 §8–12；S5 §5–8 |

**待決（不阻擋本架構）**
- Session6 §5.3 未定：每頁結果數量、「查看更多」分頁或載入更多。
- Google Maps API 金鑰與計費帳號的取得。
- v1 是否提供深色模式（S5 §8 建議 v1.1）。
