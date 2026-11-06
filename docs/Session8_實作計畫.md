# Community Information Platform — Frontend MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 以 Vue 2.7 純前端 SPA ＋ mock 資料層，實作 Sessions 1–7 定義的社區資訊平台 MVP：申報事件、查詢事件、搜尋福利活動三支流程，含 mobile-first 響應式與無障礙。

**Architecture:** 分層 `views → store(Vuex) → services → api(axios + mock adapter) → mocks(JSON)`。所有「智慧／外部」能力包在 service 介面後（`USE_MOCK` 單一開關），未來可無痛接真後端。元件不直接呼叫 axios。

**Tech Stack:** Vue 2.7、Vite（`@vitejs/plugin-vue2`）、pnpm、Element UI 2.15.x（SCSS 客製）、Vue Router 3、Vuex 3、axios ＋ axios-mock-adapter、Google Maps JS API、Web Speech API、Vitest（`@vue/test-utils` 1.x）。

**Spec:** `Session7_系統架構.md`（主）、`Session6_使用者流程設計.md`、`Session5_設計方向_Firefox風格.md`、`Session4_UX規劃.md`、`Session3_介面規劃.md`。實作者需同時閱讀本計畫與 Session7。

## Global Constraints

- **G1** 每個檔案 < 200 行；模組化（單一職責、明確邊界）。
- **G2** 元件不得直接呼叫 axios；一律經 store／service。
- **G3** 所有智慧／外部能力以 service 介面包裝，單一 `USE_MOCK` 開關；views/store 不因換真後端而改。
- **G4** 無障礙底線：對比 ≥4.5:1、`:focus-visible` 可見、觸控 ≥48px、尊重 `prefers-reduced-motion`；品牌橘 `#FF7139` 僅裝飾，不承載文字／按鈕底。
- **G5** 畫面上不得出現 Agent／API／Database／多代理等技術詞。
- **G6** mobile-first；手機（<768）底部浮動導覽＋單欄；桌機（1024+）頂部導覽＋適合處雙欄；斷點 375／768／1024／1440。
- **G7** 應用置於 `app/` 子目錄；資料層為 axios ＋ axios-mock-adapter ＋ 本地 JSON。
- **G8** 文案使用台灣繁體中文（依 S4 §6）。
- **測試指令**：`pnpm -C app test`（= `vitest run`）；單檔 `pnpm -C app exec vitest run <path>`。
- **Git**：本目錄目前不是 git repo。Task 1 含可選的 `git init`；若未啟用 git，將各任務最後的「Checkpoint」視為存檔點（略過 commit 指令）。

## Review Focus

以下為 spec 隱含、但一般 happy-path 測試不會涵蓋、最容易出事的輸入／失敗模式（最可能先踩到的排在最前）；每個都在「擁有該程式碼的任務」內加上對應測試。

1. **意圖模糊的萬用輸入**（如「最近有什麼？」）→ 必須進 P12 釐清，不得自行猜成某一流程。
2. **完全無法確認位置** → 必須降級為低精度／未確認，且**不得生成假座標**（S6 §14.4）。
3. **福利搜尋逾時／部分來源失敗** → 不白屏、不靜默失敗，須有分段回饋與出路。
4. **無可靠結果** → 顯示「找不到」，不得回傳低信心或捏造內容（S2 §22）。
5. **定位／麥克風權限被拒** → 文字輸入路徑仍完整可用，流程不受阻。

---

## File Structure

```text
app/
├── package.json                      ← 依賴與 scripts（dev/build/test）
├── vite.config.js                    ← plugin-vue2、vitest 設定
├── index.html
├── .env.local                        ← VITE_GOOGLE_MAPS_KEY（不進版控）
└── src/
    ├── main.js                       ← 建立 Vue、掛 Router/Store
    ├── App.vue                       ← 只放 <router-view>（外框由 AppShell 提供）
    ├── router/index.js               ← 路由表 + 流程護欄
    ├── store/index.js                ← Vuex root
    ├── store/modules/{report,query,welfare,location,ui}.js
    ├── api/http.js                   ← axios instance
    ├── api/mock.js                   ← axios-mock-adapter 註冊
    ├── services/intentService.js     ← USE_MOCK 開關 + classify
    ├── services/eventService.js
    ├── services/welfareService.js
    ├── services/locationService.js
    ├── services/speechService.js
    ├── services/__tests__/*.spec.js
    ├── mocks/{events,welfare,places,taxonomy}.json
    ├── mocks/scenarios.js            ← ?mock= 情境開關
    ├── views/home/HomeView.vue
    ├── views/report/{DescribeStep,LocationStep,ConfirmStep,DoneView}.vue
    ├── views/query/{EventListView,EventDetailView}.vue
    ├── views/welfare/{WelfareView,SearchInput,SearchingState,ResultList,NoResultState}.vue
    ├── views/clarify/ClarifyView.vue
    ├── components/{AppShell,TopNav,BottomNav,BigTaskCard,NlInputBox,VoiceButton,ResultCard,SourceBadge,StepIndicator,EmptyState,ErrorAlert}.vue
    ├── components/icons/*.vue
    ├── composables/useBreakpoint.js
    ├── styles/{tokens.css,element-variables.scss,element-override.scss}
    └── utils/{ranking.js,timeFilter.js,textExtract.js}
```

---

## Milestone 0 — 腳手架

### Task 1: 初始化 `app/` 專案（Vite + Vue 2.7 + Vitest）

**Files:**
- Create: `app/package.json`, `app/vite.config.js`, `app/index.html`, `app/src/main.js`, `app/src/App.vue`, `app/src/styles/tokens.css`
- Create: `app/src/__tests__/smoke.spec.js`

**Interfaces:**
- Consumes: 無
- Produces: 可用的 `pnpm -C app dev` / `pnpm -C app build` / `pnpm -C app test`；`src/styles/tokens.css` 供全站匯入。

- [ ] **Step 1: 建立 `app/package.json`**

scripts：`dev: vite`、`build: vite build`、`preview: vite preview`、`test: vitest run`。deps：`vue@^2.7.16`、`vue-router@^3.6.5`、`vuex@^3.6.2`、`element-ui@^2.15.14`、`axios@^1`。devDeps：`vite@^5`、`@vitejs/plugin-vue2@^2`、`vitest@^1`、`@vue/test-utils@^1.3`、`jsdom@^24`、`sass@^1`、`axios-mock-adapter@^1`。

- [ ] **Step 2: 建立 `app/vite.config.js` + `index.html` + `src/main.js` + `src/App.vue`**

`vite.config.js` 使用 `@vitejs/plugin-vue2`；加 `test:{ environment:'jsdom', globals:true }`。`App.vue` 先只渲染 `<h1>社區資訊平台</h1>`。

- [ ] **Step 3: 匯入 design tokens**

把 `design-system/firefox-acorn/tokens.css` 的 **Base＋Application 兩層**（第 1–153 行；**不含** `@theme`/Tailwind 段落）複製為 `app/src/styles/tokens.css`，並在 `main.js` `import './styles/tokens.css'`。

- [ ] **Step 4: 寫 smoke 測試**

```js
// app/src/__tests__/smoke.spec.js
import { mount } from '@vue/test-utils'
import App from '../App.vue'
test('App 會渲染平台名稱', () => {
  const wrapper = mount(App)
  expect(wrapper.text()).toContain('社區資訊平台')
})
```

- [ ] **Step 5: 安裝並驗證**

Run: `pnpm -C app install` → 成功安裝。
Run: `pnpm -C app test` → smoke 測試 **PASS**。
Run: `pnpm -C app build` → 產出 `dist/`，無錯誤。

- [ ] **Step 6: Checkpoint**

（可選）`git init && git add -A && git commit -m "chore: scaffold vue2 frontend"`；未用 git 則存檔即可。

---

### Task 2: 路由骨架與 AppShell（含手機／桌機導覽）

**Files:**
- Create: `app/src/router/index.js`, `app/src/components/AppShell.vue`, `app/src/components/TopNav.vue`, `app/src/components/BottomNav.vue`, `app/src/composables/useBreakpoint.js`
- Create: 路由 placeholder views：`views/home/HomeView.vue`、`views/report/DescribeStep.vue`、`views/report/LocationStep.vue`、`views/report/ConfirmStep.vue`、`views/report/DoneView.vue`、`views/query/EventListView.vue`、`views/query/EventDetailView.vue`、`views/welfare/WelfareView.vue`、`views/clarify/ClarifyView.vue`（先各只有標題）
- Modify: `app/src/main.js`, `app/src/App.vue`
- Test: `app/src/components/__tests__/nav.spec.js`

**Interfaces:**
- Consumes: Task 1 的 Vite 專案。
- Produces: 路由名稱 `home|report|report-location|report-confirm|report-done|events|event-detail|welfare|clarify`；`AppShell` 依 `useBreakpoint()` 切換 `TopNav`/`BottomNav`；導覽項目常數 `NAV_ITEMS = [{to:'/report',label:'回報問題'},{to:'/events',label:'看看附近事件'},{to:'/welfare',label:'找福利和活動'}]`。

- [ ] **Step 1: 寫導覽測試（失敗）**

```js
// app/src/components/__tests__/nav.spec.js
import { shallowMount } from '@vue/test-utils'
import BottomNav from '../BottomNav.vue'
test('底部導覽有三個項目且文字正確', () => {
  const w = shallowMount(BottomNav, { stubs: ['router-link'] })
  expect(w.findAll('[data-nav-item]').length).toBe(3)
  expect(w.text()).toContain('找福利和活動')
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `pnpm -C app exec vitest run src/components/__tests__/nav.spec.js` → FAIL（找不到元件）。

- [ ] **Step 3: 實作路由與導覽**

建 `router/index.js`（history 模式、上表路由）；`NAV_ITEMS` 放 `router/index.js` 並 export 供兩個 Nav 共用。`useBreakpoint.js` 用 `matchMedia('(min-width:1024px)')` 回傳 `isDesktop`。`AppShell.vue`：`<TopNav v-if=isDesktop/><slot/><BottomNav v-else/>`。`App.vue` 用 `AppShell` 包 `<router-view>`。`TopNav`/`BottomNav` 都渲染 `NAV_ITEMS`，每項加 `data-nav-item`。

- [ ] **Step 4: 執行測試確認通過**

Run: `pnpm -C app exec vitest run src/components/__tests__/nav.spec.js` → PASS。

- [ ] **Step 5: 手動驗證**

Run: `pnpm -C app dev`；逐一點擊三個導覽項目確認路由切換、畫面標題正確。

- [ ] **Step 6: Checkpoint**（git 或存檔）

---

### Task 3: Element UI ＋ Firefox 主題 ＋ 響應式樣式基底

**Files:**
- Create: `app/src/styles/element-variables.scss`, `app/src/styles/element-override.scss`
- Modify: `app/src/main.js`（引入 ElementUI 與樣式）
- Test: `app/src/styles/__tests__/tokens.spec.js`

**Interfaces:**
- Consumes: Task 1 的 `tokens.css`。
- Produces: 全域主題：`$--color-primary: #a44900`；按鈕 ≥52px、輸入 ≥48px、字級 ≥18px；`:focus-visible` 用 `--ring`；手機 `scroll-padding-bottom`。

- [ ] **Step 1: 寫 token 測試（失敗）**

```js
// app/src/styles/__tests__/tokens.spec.js
import fs from 'node:fs'
const css = fs.readFileSync(new URL('../tokens.css', import.meta.url), 'utf8')
test('包含 Firefox 暖色 primary 與 radius tokens', () => {
  expect(css).toContain('--primary')
  expect(css).toContain('#a44900')
  expect(css).toContain('--radius-card')
})
```

- [ ] **Step 2: 執行確認失敗或通過**

Run: `pnpm -C app exec vitest run src/styles/__tests__/tokens.spec.js`。若 Task 1 已複製 tokens 則此測試應 PASS；若 FAIL 代表 tokens 未正確複製，修正後再跑。

- [ ] **Step 3: 實作 Element UI 主題**

`element-variables.scss`：設 `$--color-primary:#a44900; $--border-radius-base:12px;` 等，`@import "~element-ui/packages/theme-chalk/src/index";`（或等價 SCSS 客製）。`element-override.scss`：覆寫按鈕高度/圓角、input 高度、`:focus-visible outline:2px solid var(--ring)`、`html{font-size:18px}`、`body{line-height:1.75}`、`*{scroll-padding-bottom:5.5rem}`。`main.js` 依序引入 tokens → element-variables → element-override。

- [ ] **Step 4: 驗證**

Run: `pnpm -C app build` → 成功（確認 sass 編譯通過）。

- [ ] **Step 5: Checkpoint**（git 或存檔）

---

### Task 4: axios ＋ mock adapter ＋ JSON 種子資料

**Files:**
- Create: `app/src/api/http.js`, `app/src/api/mock.js`, `app/src/mocks/scenarios.js`, `app/src/mocks/events.json`, `app/src/mocks/welfare.json`, `app/src/mocks/places.json`, `app/src/mocks/taxonomy.json`
- Test: `app/src/api/__tests__/http.spec.js`

**Interfaces:**
- Consumes: `axios`。
- Produces: `http`（axios instance，`baseURL:'/api'`）；`setupMock(http)`（在非 production 啟用）；`scenario()`（讀 `location.search` 的 `mock` 參數 → `null|'empty'|'timeout'|'partial'|'error'|'ambiguous'`）。JSON 皆為陣列或物件，欄位見各 service 任務。

- [ ] **Step 1: 寫測試（失敗）**

```js
// app/src/api/__tests__/http.spec.js
import http from '../http'
import { setupMock } from '../mock'
setupMock(http)
test('GET /api/health 經 mock 回 ok', async () => {
  const { data } = await http.get('/api/health')
  expect(data.ok).toBe(true)
})
```

- [ ] **Step 2: 執行確認失敗**

Run: `pnpm -C app exec vitest run src/api/__tests__/http.spec.js` → FAIL。

- [ ] **Step 3: 實作 http 與 mock**

`http.js`：`axios.create({ baseURL:'/api', timeout:15000 })`。`mock.js`：`new MockAdapter(http, { delayResponse: 600 })`，註冊 `/api/health` 等（後續任務再擴充）；`scenarios.js`：解析 `?mock=`。

- [ ] **Step 4: 執行確認通過**

Run: 同上 → PASS。

- [ ] **Step 5: Checkpoint**（git 或存檔）

---

## Milestone 1 — 共用元件

### Task 5: 線性 SVG 圖示元件

**Files:**
- Create: `app/src/components/icons/{House,Flag,MapPin,Gift,ArrowLeft,Calendar,Microphone}.vue`
- Test: `app/src/components/icons/__tests__/icons.spec.js`

**Interfaces:**
- Consumes: 無。
- Produces: 每個元件輸出 `<svg viewBox="0 0 24 24" width height stroke="currentColor" fill="none" stroke-width="1.75">`；props：`size`（預設 24）。`aria-hidden` 由使用端決定。

- [ ] **Step 1: 寫測試（失敗）**

```js
import { shallowMount } from '@vue/test-utils'
import House from '../House.vue'
test('House 輸出 svg 且預設 24px', () => {
  const w = shallowMount(House)
  expect(w.find('svg').exists()).toBe(true)
  expect(w.find('svg').attributes('width')).toBe('24')
})
```

- [ ] **Step 2: 執行確認失敗** → Run: `pnpm -C app exec vitest run src/components/icons` → FAIL。
- [ ] **Step 3: 實作 7 個圖示**（線性、統一 `stroke-width`；路徑可用 Phosphor 開源 SVG path）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: Checkpoint**（git 或存檔）

---

### Task 6: 輸入元件（BigTaskCard / NlInputBox / VoiceButton）＋ 語音服務

**Files:**
- Create: `app/src/components/BigTaskCard.vue`, `app/src/components/NlInputBox.vue`, `app/src/components/VoiceButton.vue`, `app/src/services/speechService.js`
- Test: `app/src/services/__tests__/speechService.spec.js`

**Interfaces:**
- Consumes: Task 5 圖示。
- Produces:
  - `BigTaskCard` props `{icon, title, desc, to}`；整卡可點。
  - `NlInputBox` props `{value}`，events `input`、`voice`；含可見 `<label>`；multi-line ≥120px。
  - `VoiceButton` props `{supported}`，events `result(text)`、`error`。
  - `speechService`：`isSupported()`、`start({onResult,onError,lang='zh-TW'})`、`stop()`；不支援時 `start` 立即 `onError` 但**不丟例外**。

- [ ] **Step 1: 寫測試（失敗）**

```js
// speechService.spec.js
import speechService from '../speechService'
test('不支援時 start 會回報錯誤但不拋例外', () => {
  let err = null
  expect(() => speechService.start({ onResult(){}, onError(e){ err = e } })).not.toThrow()
  if (!speechService.isSupported()) expect(err).toBeTruthy()
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作 `speechService` 與三個元件**（`NlInputBox` 的語音按鈕在使用者點擊時才呼叫 `start`）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: Checkpoint**（git 或存檔）

---

### Task 7: 顯示元件（ResultCard / SourceBadge / StepIndicator / EmptyState / ErrorAlert）

**Files:**
- Create: `app/src/components/{ResultCard,SourceBadge,StepIndicator,EmptyState,ErrorAlert}.vue`
- Test: `app/src/components/__tests__/resultCard.spec.js`

**Interfaces:**
- Consumes: Task 5 圖示。
- Produces:
  - `ResultCard` props `{title, dateText, placeText, source, publishedAt, url}`；渲染來源／日期／原始連結。
  - `SourceBadge` props `{unit, publishedAt}`。
  - `StepIndicator` props `{current, total}` → `步驟 X/Y`。
  - `EmptyState` props `{message}`，slot `action`（下一步按鈕）。
  - `ErrorAlert` props `{message}`；根元素 `role="alert"`、`tabindex="-1"`。

- [ ] **Step 1: 寫測試（失敗）**

```js
import { mount } from '@vue/test-utils'
import ResultCard from '../ResultCard.vue'
test('ResultCard 顯示來源與原始連結', () => {
  const w = mount(ResultCard, { propsData: { title:'健康檢查', source:'XX區公所', publishedAt:'10/01', url:'https://x' } })
  expect(w.text()).toContain('XX區公所')
  expect(w.find('a[href="https://x"]').exists()).toBe(true)
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作五個元件**（`ErrorAlert` 加 `role="alert"`）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: Checkpoint**（git 或存檔）

---

## Milestone 2 — 核心邏輯 Services（TDD）

### Task 8: 工具函式（textExtract / ranking / timeFilter）

**Files:**
- Create: `app/src/utils/textExtract.js`, `app/src/utils/ranking.js`, `app/src/utils/timeFilter.js`
- Test: `app/src/utils/__tests__/textExtract.spec.js`, `app/src/utils/__tests__/timeFilter.spec.js`

**Interfaces:**
- Consumes: `mocks/taxonomy.json`（事件類型關鍵詞）。
- Produces:
  - `textExtract.extract(text) → { eventType:string|null, locationText:string|null, timeText:string|null }`
  - `timeFilter.isExpired(dateStr, now=new Date()) → boolean`
  - `timeFilter.inRange(dateStr, range, now) → boolean`（range: `today|week|month|all`）
  - `ranking.sortByRelevance(items, params) → items`（先比地區符合、再比時間新近）

- [ ] **Step 1: 寫測試（失敗）**

```js
// textExtract.spec.js
import { extract } from '../textExtract'
test('抽出坑洞與地標', () => {
  const r = extract('中華路全家旁邊有一個坑洞')
  expect(r.eventType).toBe('坑洞')
  expect(r.locationText).toContain('中華路')
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作三個 util**（純函式、無副作用）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: Checkpoint**（git 或存檔）

---

### Task 9: `intentService`（意圖分流）

**Files:**
- Create: `app/src/services/intentService.js`
- Modify: `app/src/api/mock.js`（註冊 `/api/intent`）
- Test: `app/src/services/__tests__/intentService.spec.js`

**Interfaces:**
- Consumes: `http`（Task 4）、`textExtract`（Task 8）。
- Produces: `classify(text) → Promise<{ intent:'report'|'query'|'welfare'|'ambiguous', confidence:number, extracted:object }>`；`USE_MOCK=true` 時走規則。

- [ ] **Step 1: 寫測試（失敗，覆蓋 Review Focus #1）**

```js
import intentService from '../intentService'
it.each([
  ['中華路全家旁邊有一個坑洞', 'report'],
  ['附近有沒有積水', 'query'],
  ['這個月有什麼老人活動', 'welfare'],
  ['最近有什麼', 'ambiguous'],
])('%s → %s', async (text, intent) => {
  const r = await intentService.classify(text)
  expect(r.intent).toBe(intent)
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作規則分類**（出現具體事件描述→report；「有沒有／最近…嗎」→query；補助/活動/福利/課程/健康檢查→welfare；其餘→ambiguous）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: Checkpoint**（git 或存檔）

---

### Task 10: `locationService`（含 S6 §14.4 決策）

**Files:**
- Create: `app/src/services/locationService.js`
- Modify: `app/src/api/mock.js`（註冊 `/api/places`）
- Test: `app/src/services/__tests__/locationService.spec.js`

**Interfaces:**
- Consumes: `http`、`mocks/places.json`。
- Produces:
  - `geocode(text) → Promise<Candidate[]>`，`Candidate = {id, name, address, lat, lng, score}`
  - `reverse(lat,lng) → Promise<string>`
  - `decide({ onSite:boolean, permissionGranted:boolean, candidates:Candidate[], wantsMap:boolean }) → { mode:'gps'|'candidate'|'map-confirm'|'low-precision'|'unconfirmed', coordinates:{lat,lng}|null }`

- [ ] **Step 1: 寫測試（失敗，覆蓋 Review Focus #2）**

```js
import { decide } from '../locationService'
test('完全無法確認且不使用地圖 → 未確認且無假座標', () => {
  const r = decide({ onSite:false, permissionGranted:false, candidates:[], wantsMap:false })
  expect(r.mode).toBe('unconfirmed')
  expect(r.coordinates).toBeNull()
})
test('多候選 → 地圖確認', () => {
  const r = decide({ onSite:false, permissionGranted:false, candidates:[{id:1},{id:2}], wantsMap:false })
  expect(r.mode).toBe('map-confirm')
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作 `decide` 與 `geocode`/`reverse`**（`geocode` mock 讀 places.json 計分；`decide` 依 onSite/permission/candidates 收斂模式；**任何路徑都不得合成座標**）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: Checkpoint**（git 或存檔）

---

### Task 11: `eventService`（申報／查詢／詳情）

**Files:**
- Create: `app/src/services/eventService.js`
- Modify: `app/src/api/mock.js`（`/api/events`，GET 列表／POST 新增／GET :id）
- Test: `app/src/services/__tests__/eventService.spec.js`

**Interfaces:**
- Consumes: `http`、`mocks/events.json`、`textExtract`、`timeFilter`。
- Produces:
  - `list({ region, time, type, status }) → Promise<Event[]>`（預設近期，非全部）
  - `get(id) → Promise<Event|null>`（不存在代表已下架）
  - `report(draft) → Promise<Event>`
  - `Event = { id, type, title, placeText, coordinates, timeText, status, description, photo, reportedAt }`

- [ ] **Step 1: 寫測試（失敗，覆蓋 Review Focus #4 空狀態）**

```js
import eventService from '../eventService'
test('篩選不存在的類型 → 空陣列', async () => {
  const r = await eventService.list({ type:'不存在的類型' })
  expect(r).toEqual([])
})
test('get 不存在的 id → null', async () => {
  expect(await eventService.get('nope')).toBeNull()
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作三個方法**（`report` 產生 id 並把新事件加入 mock 回應；`list` 依 filters 過濾後排序）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: Checkpoint**（git 或存檔）

---

### Task 12: `welfareService`（去重／時效／排序／無可靠結果）

**Files:**
- Create: `app/src/services/welfareService.js`
- Modify: `app/src/api/mock.js`（`/api/welfare`）
- Test: `app/src/services/__tests__/welfareService.spec.js`

**Interfaces:**
- Consumes: `http`、`mocks/welfare.json`、`ranking`、`timeFilter`。
- Produces: `search({ type, target, time, region }) → Promise<{ results:Item[], partial:boolean, failedSources:string[] }>`；`Item = {id,title,source,publishedAt,eventDate,url}`。**低信心不呈現**；全部不可靠 → `results:[]`。

- [ ] **Step 1: 寫測試（失敗，覆蓋 Review Focus #3、#4）**

```js
import welfareService from '../welfareService'
test('去除重複且過濾已過期活動', async () => {
  const { results } = await welfareService.search({ type:'活動' })
  const keys = results.map(r => r.title + r.eventDate)
  expect(new Set(keys).size).toBe(keys.length)
})
test('全部不可靠時 results 為空', async () => {
  const { results } = await welfareService.search({ type:'__none__' })
  expect(results).toEqual([])
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作 search**（讀 JSON → 時效過濾 → 去重（title+eventDate）→ 相關性排序 → 標記 partial/failedSources）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: Checkpoint**（git 或存檔）

---

### Task 13: Mock 情境開關接線（empty / timeout / partial / error / ambiguous）

**Files:**
- Modify: `app/src/api/mock.js`, `app/src/services/{welfare,intent,location}Service.js`
- Test: `app/src/mocks/__tests__/scenarios.spec.js`

**Interfaces:**
- Consumes: `scenarios.js`（Task 4）。
- Produces: 當 `?mock=timeout` 時 `welfareService.search` 延遲 ≥15s（或回傳 `{timeout:true}`）；`?mock=empty` → 空結果；`?mock=partial` → `partial:true, failedSources:[...]`；`?mock=error` → 服務回 500 讓呼叫端走錯誤狀態；`?mock=ambiguous` → `intentService` 一律回 `ambiguous`。

- [ ] **Step 1: 寫測試（失敗，覆蓋 Review Focus #3）**

```js
import { scenario } from '../scenarios'
test('scenario 解析查詢字串', () => {
  expect(['empty','timeout','partial','error','ambiguous',null]).toContain(scenario())
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 在 mock 與 service 中接上情境分支**。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: Checkpoint**（git 或存檔）

---

## Milestone 3 — 流程畫面

### Task 14: P0 首頁 Hub ＋ 意圖分流

**Files:**
- Create: `app/src/store/modules/intent.js`, `app/src/store/modules/report.js`（初版）
- Modify: `app/src/store/index.js`, `app/src/views/home/HomeView.vue`, `app/src/router/index.js`（移除 P0 placeholder）
- Test: `app/src/store/__tests__/intent.spec.js`

**Interfaces:**
- Consumes: `intentService`（Task 9）、`BigTaskCard`／`NlInputBox`（Task 6）。
- Produces: `intent.routeFromText(text) → Promise<{ name, params?, query? }>` 映射：`report→'report'`、`query→'events'`、`welfare→'welfare'`、`ambiguous→'clarify'`；`report` 時帶入 `draft.description`。HomeView 三卡片分別 `router-link` 到 `/report`、`/events`、`/welfare`。

- [ ] **Step 1: 寫測試（失敗，覆蓋 Review Focus #1）**

```js
import { createLocalVue } from '@vue/test-utils'
import Vuex from 'vuex'
import intent from '../modules/intent'
const localVue = createLocalVue(); localVue.use(Vuex)
test('ambiguous 導向 clarify', async () => {
  const store = new Vuex.Store({ modules: { intent } })
  const r = await store.dispatch('intent/routeFromText', '最近有什麼')
  expect(r.name).toBe('clarify')
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作 intent 模組與 HomeView**（萬用輸入送出 → `routeFromText` → `$router.push`）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: 手動驗證** 四種輸入各自導向正確頁面。
- [ ] **Step 6: Checkpoint**（git or 存檔）

---

### Task 15: P1 申報：描述事件

**Files:**
- Create: `app/src/store/modules/report.js`（補 `describe`）、`app/src/router/index.js`（report 護欄）
- Modify: `app/src/views/report/DescribeStep.vue`
- Test: `app/src/store/__tests__/report.spec.js`

**Interfaces:**
- Consumes: `textExtract`（Task 8）、`NlInputBox`／`StepIndicator`（Task 6/7）。
- Produces: `report.describe(text)`：呼叫 `textExtract` 填入 `draft{description, category}`；回傳 `{ missing:string[] }`（缺哪一項）。`report` state `{ draft:{description,category,location,time,photo}, step, status }`。護欄：`/report/location` 無 `draft.description` → 導回 `/report`。

- [ ] **Step 1: 寫測試（失敗）**

```js
test('describe 解析事件並回報缺少欄位', async () => {
  const store = new Vuex.Store({ modules: { report } })
  const r = await store.dispatch('report/describe', '中華路有一個坑洞')
  expect(store.state.report.draft.category).toBe('坑洞')
  expect(r.missing).toContain('location')
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作 `describe` 與 DescribeStep 畫面**（單一輸入框＋語音；空輸入時禁用下一步）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: Checkpoint**（git 或存檔）

---

### Task 16: P2 申報：確認位置

**Files:**
- Create: `app/src/store/modules/location.js`, `app/src/components/MapPicker.vue`（Google Maps 封裝）
- Modify: `app/src/views/report/LocationStep.vue`
- Test: `app/src/store/__tests__/location.spec.js`

**Interfaces:**
- Consumes: `locationService`（Task 10）、`StepIndicator`。
- Produces: `location.requestCurrent()`（Geolocation，拒權不丟例外）、`location.resolveText(text)`、`location.pickCandidate(id)`、`location.fallback(mode)`；state `{ current, candidates, selected, permission }`。`MapPicker` props `{lat,lng,marker}`，events `pick(lat,lng)`；拖曳**非唯一**操作（另提供「點選」與鍵盤移動）。
- **位置決策**由 `locationService.decide` 決定模式；`unconfirmed` 時允許「先送出」但 `draft.location` 記 `{ mode:'unconfirmed', text:<原話> }`。

- [ ] **Step 1: 寫測試（失敗，覆蓋 Review Focus #2）**

```js
test('拒權後仍可文字描述，且不會填入假座標', () => {
  const store = new Vuex.Store({ modules: { location } })
  store.commit('location/setPermission', 'denied')
  expect(store.state.location.selected).toBeNull()
  expect(store.getters['location/hasCoordinates']).toBe(false)
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作 location 模組、MapPicker、LocationStep**（含 P2 三情境按鈕與 unconfirmed 出路）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: 手動驗證** 定位成功／拒權／多候選／完全無法確認四條路徑。
- [ ] **Step 6: Checkpoint**（git 或存檔）

---

### Task 17: P3 摘要確認 ＋ P4 完成

**Files:**
- Modify: `app/src/store/modules/report.js`（`submit`）、`app/src/views/report/ConfirmStep.vue`, `app/src/views/report/DoneView.vue`
- Test: `app/src/store/__tests__/reportSubmit.spec.js`

**Interfaces:**
- Consumes: `eventService.report`（Task 11）、`StepIndicator`、`ErrorAlert`。
- Produces: `report.submit()`：`status='submitting'`→`eventService.report(draft)`→`status='submitted'`；失敗 → `status='error'` 並保留 draft。`report.reset()` 清空（回首頁時）。ConfirmStep 的「修改」導回對應步驟且不丟資料。

- [ ] **Step 1: 寫測試（失敗）**

```js
test('送出成功後 status 為 submitted 且帶回 id', async () => {
  const store = new Vuex.Store({ modules: { report } })
  await store.dispatch('report/submit')
  expect(store.state.report.status).toBe('submitted')
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作 `submit`／`reset` 與兩個畫面**（送出按鈕 loading+disabled；P4 顯示「已收到你的回報…」）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: Checkpoint**（git 或存檔）

---

### Task 18: P5 查詢目前事件

**Files:**
- Create: `app/src/store/modules/query.js`
- Modify: `app/src/views/query/EventListView.vue`
- Test: `app/src/store/__tests__/query.spec.js`

**Interfaces:**
- Consumes: `eventService.list`（Task 11）、`ResultCard`／`EmptyState`。
- Produces: `query.loadEvents()`、`query.setFilter(partial)`、`query.clearFilters()`；state `{ region, filters:{time,type,status}, events, loading, error }`。空結果 → `EmptyState` 含 slot action（回報一個問題 → `/report`）。

- [ ] **Step 1: 寫測試（失敗，覆蓋 Review Focus #4）**

```js
test('loadEvents 空結果時 events 為空且非錯誤', async () => {
  const store = new Vuex.Store({ modules: { query } })
  await store.dispatch('query/loadEvents')
  expect(Array.isArray(store.state.query.events)).toBe(true)
  expect(store.state.query.error).toBeFalsy()
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作 query 模組與畫面**（預設近期；chips 篩選；為空時「清除篩選」）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: Checkpoint**（git 或存檔）

---

### Task 19: P7 事件詳情

**Files:**
- Modify: `app/src/views/query/EventDetailView.vue`, `app/src/store/modules/query.js`（`loadDetail`）
- Test: `app/src/store/__tests__/queryDetail.spec.js`

**Interfaces:**
- Consumes: `eventService.get`（Task 11）。
- Produces: `query.loadDetail(id)`；`null` → state `detailMissing=true`（顯示「此事件已移除」＋回列表）。

- [ ] **Step 1: 寫測試（失敗）**

```js
test('loadDetail 找不到 → detailMissing 為 true', async () => {
  const store = new Vuex.Store({ modules: { query } })
  await store.dispatch('query/loadDetail', 'nope')
  expect(store.state.query.detailMissing).toBe(true)
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作 `loadDetail` 與詳情畫面**（地點/時間/狀態/描述/照片；不顯示申報者個資）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: Checkpoint**（git 或存檔）

---

### Task 20: P8–P11 搜尋福利與活動

**Files:**
- Create: `app/src/store/modules/welfare.js`
- Modify: `app/src/views/welfare/{WelfareView,SearchInput,SearchingState,ResultList,NoResultState}.vue`
- Test: `app/src/store/__tests__/welfare.spec.js`

**Interfaces:**
- Consumes: `welfareService.search`（Task 12）、`ResultCard`／`EmptyState`／`ErrorAlert`。
- Produces: `welfare.search(keyword)`：`phase` 依序 `loading→processing→done`，配合三段式文案；`welfare.retry()`、`welfare.widen()`；state `{ keyword, phase, results, partial, failedSources, timeout, error }`。`phase==='done' && results.length===0` → `NoResultState`；`partial` → 顯示略過提示；逾時（Task 13）→ 顯示「繼續等待／重新搜尋」。
- 三段式文案（G8）：`正在理解你的問題…`→`正在搜尋政府與社區資訊…`→`正在整理結果…`。

- [ ] **Step 1: 寫測試（失敗，覆蓋 Review Focus #3）**

```js
vi.useFakeTimers()
test('search 完成後 phase 為 done', async () => {
  const store = new Vuex.Store({ modules: { welfare } })
  await store.dispatch('welfare/search', '老人活動')
  expect(store.state.welfare.phase).toBe('done')
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作 welfare 模組與四個子畫面**（依 phase 切換；`ResultList` 每筆含來源／發布日／活動日／原始連結）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: 手動驗證** `?mock=timeout`／`empty`／`partial`／`error` 四個狀態。
- [ ] **Step 6: Checkpoint**（git 或存檔）

---

### Task 21: P12 模糊需求釐清

**Files:**
- Modify: `app/src/views/clarify/ClarifyView.vue`, `app/src/store/modules/intent.js`（`resolveChoice`）
- Test: `app/src/store/__tests__/intent.spec.js`（追加）

**Interfaces:**
- Consumes: `intent.routeFromText`（Task 14）。
- Produces: `intent.resolveChoice(choice)`：`choice='event'` → 對事件再判斷 `report`/`query`（S6 §1.2）→ 回傳 route；`choice='welfare'` → `{ name:'welfare' }`；事件無法判斷 → `{ name:'clarify-event' }`（事件入口二選一，畫面同頁第二層）。

- [ ] **Step 1: 寫測試（失敗）**

```js
test('選 welfare 直接導向福利搜尋', async () => {
  const store = new Vuex.Store({ modules: { intent } })
  await expect(store.dispatch('intent/resolveChoice', 'welfare')).resolves.toEqual({ name:'welfare' })
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作 `resolveChoice` 與 ClarifyView**（兩層大按鈕；只在必要時出現）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: Checkpoint**（git 或存檔）

---

## Milestone 4 — 響應式與無障礙驗收

### Task 22: 桌面版型（頂部導覽＋雙欄）

**Files:**
- Modify: `app/src/components/{AppShell,TopNav}.vue`, `app/src/composables/useBreakpoint.js`, `app/src/views/{home/HomeView,query/EventListView,welfare/WelfareView}.vue`, `app/src/styles/element-override.scss`
- Test: `app/src/composables/__tests__/useBreakpoint.spec.js`

**Interfaces:**
- Consumes: Task 2 的導覽。
- Produces: `isDesktop`（`matchMedia('(min-width:1024px)')`，可監聽變化）；桌機時頂部導覽 `sticky`、內容 `max-width` 收斂；HomeView 卡片、EventListView／ResultList 在 1024+ 以 **CSS grid 兩欄**呈現。手機維持單欄＋底部浮動導覽。

- [ ] **Step 1: 寫測試（失敗）**

```js
import { useBreakpoint } from '../useBreakpoint'
test('useBreakpoint 回傳 isDesktop 布林', () => {
  expect(typeof useBreakpoint().isDesktop).toBe('boolean')
})
```

- [ ] **Step 2: 執行確認失敗** → FAIL。
- [ ] **Step 3: 實作桌機版型與雙欄**（media query；不影響手機）。
- [ ] **Step 4: 執行確認通過** → PASS。
- [ ] **Step 5: 手動驗證** 375／768／1024／1440 四個斷點版面。
- [ ] **Step 6: Checkpoint**（git 或存檔）

---

### Task 23: 無障礙與 375px 手動驗收

**Files:**
- Modify: 依檢查結果微調各元件／樣式
- Create: `app/ACCEPTANCE.md`（逐項打勾紀錄）

**Interfaces:**
- Consumes: 全體。
- Produces: 通過 S4 §12 交付檢查清單與 G4 的驗收紀錄。

- [ ] **Step 1: 逐項驗收（依 S4 §12）**：每畫面 Default/Loading/Empty/Error/Partial/Success；空／錯狀態有下一步；一次只問一欄；位置有文字替代路徑且地圖有非拖曳操作；`scroll-padding-bottom`；每筆結果附來源／日期／連結；等待分段回饋；文案 375px 不破版；對比 ≥4.5:1；focus 可見；觸控 ≥48px；`prefers-reduced-motion`；畫面無技術詞（G5）；語音／定位被拒仍可用文字。
- [ ] **Step 2: 鍵盤走查** 以 Tab 走完三支流程，確認 focus 不被導覽遮住。
- [ ] **Step 3: 記錄** 於 `app/ACCEPTANCE.md` 打勾並註明未過項目與後續。
- [ ] **Step 4: Checkpoint**（git 或存檔）

---

## 執行說明

- **依賴順序**：Milestone 0 → 1 → 2 → 3 → 4。Task 內若有未定義介面，以「Interfaces」區塊為準；跨任務只依賴前序任務的 Produces。
- **每任務完成即跑**：`pnpm -C app test` 全綠再進下一任務。
- **不確定時**：回到 `Session7_系統架構.md` 與 `Session6_使用者流程設計.md` 對照，不自行新增未定義功能。
- **範圍外**：真後端、真 LLM、真 multi-agent、真網頁檢索（見 Session7 附錄 A）。
