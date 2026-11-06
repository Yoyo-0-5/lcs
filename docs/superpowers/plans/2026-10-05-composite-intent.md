# 複合需求入口＋移除釐清 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 讓首頁萬用輸入列能處理「一句話多個任務」（複合需求）並以 `/results` 分區呈現結果，同時移除釐清畫面（「你是想找哪一種」）。

**Architecture:** 在 `intentRules` 新增純函式 `parseIntents`（切子句→分類→彙整不重複意圖）；`intent/routeFromText` 依 0／1／2+ 意圖決定導向（單一意圖直達、其餘進 `/results`）；新增 `ResultsView` 分區呈現事件／福利結果並提供回報入口與無法判斷時的提示。

**Tech Stack:** Vue 2.7（Options API）、Vue Router 3、Vuex 3、Vite 5、Vitest 1 + @vue/test-utils 1。

**Spec:** `docs/superpowers/specs/2026-10-05-composite-intent-design.md`

## Global Constraints

- 規則驅動：多意圖以標點／連接詞切分 + 既有 `classifyText`，不做真語意模型。
- 單一意圖**不經** `/results`，維持既有直達體驗。
- `/results` 以 `?q=` 攜帶原話（可深連結、可重整）。
- 回報意圖在複合中**只顯示入口卡**，不自動開始流程。
- 畫面不得出現 Agent／AI／API 等技術詞（G5）。
- 沿用入口基準 token；≥48px、WCAG AA、可見焦點、`prefers-reduced-motion`。
- 測試指令：`cd app; npm test`。

## Review Focus

1. **無法判斷的輸入** → 進 `/results` 顯示提示＋三個入口，不崩潰。（Task 3）
2. **複合輸入** → `/results` 同時呈現事件與福利兩區。（Task 3）
3. **單一意圖** → 不經 `/results`，直接進對應流程。（Task 2）
4. **子句切分**：標點與連接詞都能切；同意圖只留一次。（Task 1）
5. **回報意圖在複合中**只顯示入口、不自動開始流程。（Task 3）

---

### Task 1: 多意圖解析 parseIntents

**Files:**
- Modify: `app/src/services/intentRules.js`
- Create: `app/src/services/__tests__/intentRules.spec.js`

**Interfaces:**
- Produces: `parseIntents(text) -> Array<{ intent: 'report'|'query'|'welfare', text: string }>`（依出現順序、不重複意圖）。

- [ ] **Step 1: 寫失敗測試**

建立 `app/src/services/__tests__/intentRules.spec.js`：

```js
import { parseIntents, classifyText } from '../intentRules'

test('單一意圖：只有一個子句', () => {
  expect(parseIntents('附近有沒有積水')).toEqual([
    { intent: 'query', text: '附近有沒有積水' },
  ])
})

test('以標點切分：一句話多個任務', () => {
  expect(parseIntents('附近有沒有積水，這個月有什麼老人活動')).toEqual([
    { intent: 'query', text: '附近有沒有積水' },
    { intent: 'welfare', text: '這個月有什麼老人活動' },
  ])
})

test('以連接詞切分（順便）', () => {
  expect(parseIntents('中華路有坑洞 順便 這個月有什麼老人活動')).toEqual([
    { intent: 'report', text: '中華路有坑洞' },
    { intent: 'welfare', text: '這個月有什麼老人活動' },
  ])
})

test('無法判斷回空陣列', () => {
  expect(parseIntents('最近有什麼')).toEqual([])
})

test('同意圖只留一次（保留第一個子句）', () => {
  expect(parseIntents('附近有沒有積水，有沒有路燈')).toEqual([
    { intent: 'query', text: '附近有沒有積水' },
  ])
})

test('classifyText 行為不變', () => {
  expect(classifyText('這個月有什麼老人活動').intent).toBe('welfare')
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/services/__tests__/intentRules.spec.js`
Expected: FAIL（`parseIntents` 未定義）

- [ ] **Step 3: 在 `intentRules.js` 新增 `parseIntents`**

- 切分：`text.split(/[，、,。；;\n]+|順便|還有|以及|另外|也要|然後/)`，trim 後濾掉空字串。
- 逐句 `classifyText`；略過 `ambiguous`；以 `Set` 去重意圖（保留第一句）。
- 匯出 `parseIntents`。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/services/__tests__/intentRules.spec.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add app/src/services/intentRules.js app/src/services/__tests__/intentRules.spec.js
git commit -m "feat(intent): parse composite requests into multiple intents"
```

---

### Task 2: intent store 多意圖分流（移除 resolveChoice）

**Files:**
- Modify: `app/src/store/modules/intent.js`
- Modify: `app/src/store/__tests__/intent.spec.js`

**Interfaces:**
- Consumes: `parseIntents`（Task 1）。
- Produces: `routeFromText` 依意圖數回傳：
  - 0 → `{ name: 'results', query: { q: text } }`
  - 1 → `report` 時 `commit report/setDescription` 並回 `{ name: 'report' }`；`query` → `{ name: 'events', query: { q: clause } }`；`welfare` → `{ name: 'welfare', query: { q: clause } }`
  - 2+ → `{ name: 'results', query: { q: text } }`

- [ ] **Step 1: 寫失敗測試**

改寫 `intent.spec.js` 的 `routeFromText` 段，並刪除整個 `resolveChoice` 段：

```js
describe('intent.routeFromText', () => {
  test('無法判斷 → 進 results（不再進 clarify）', async () => {
    const store = makeStore()
    await expect(store.dispatch('intent/routeFromText', '最近有什麼')).resolves.toEqual({
      name: 'results',
      query: { q: '最近有什麼' },
    })
  })

  test('單一 query → 直接進 events 並帶上原話', async () => {
    const store = makeStore()
    await expect(
      store.dispatch('intent/routeFromText', '附近有沒有積水'),
    ).resolves.toEqual({ name: 'events', query: { q: '附近有沒有積水' } })
  })

  test('單一 welfare → 直接進 welfare 並帶上原話', async () => {
    const store = makeStore()
    await expect(
      store.dispatch('intent/routeFromText', '這個月有什麼老人活動'),
    ).resolves.toEqual({ name: 'welfare', query: { q: '這個月有什麼老人活動' } })
  })

  test('單一 report → 帶入草稿並進 report', async () => {
    const store = makeStore()
    const text = '中華路全家旁邊有一個坑洞'
    await expect(store.dispatch('intent/routeFromText', text)).resolves.toEqual({ name: 'report' })
    expect(store.state.report.draft.description).toBe(text)
  })

  test('複合（多意圖）→ 進 results 並帶上原話', async () => {
    const store = makeStore()
    const text = '附近有沒有積水，這個月有什麼老人活動'
    await expect(store.dispatch('intent/routeFromText', text)).resolves.toEqual({
      name: 'results',
      query: { q: text },
    })
  })
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/store/__tests__/intent.spec.js`
Expected: FAIL（仍導向 clarify／無 results）

- [ ] **Step 3: 改寫 `store/modules/intent.js`**

- 匯入 `parseIntents`；`routeFromText` 依 §4.2 規則實作。
- `remember` 記錄 `{ text, intents }`。
- 刪除 `ROUTE_BY_INTENT`、`TEXT_CARRYING_INTENTS`、`resolveChoice`。保留 `import intentService` 若仍需要（不再需要則移除）。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/store/__tests__/intent.spec.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add app/src/store/modules/intent.js app/src/store/__tests__/intent.spec.js
git commit -m "feat(intent): composite-aware routing, drop clarify action"
```

---

### Task 3: 綜合結果頁與移除釐清

**Files:**
- Create: `app/src/views/results/ResultsView.vue`
- Create: `app/src/views/results/__tests__/resultsView.spec.js`
- Modify: `app/src/router/index.js`（新增 `/results`、移除 `/clarify`）
- Delete: `app/src/views/clarify/ClarifyView.vue`、`app/src/views/clarify/__tests__/clarifyView.spec.js`

**Interfaces:**
- Consumes: `parseIntents`（Task 1）、`query/searchFromText`、`welfare/search`、既有 `ResultCard`／`ResultList`／`NoResultState`／`SearchingState`／`ErrorAlert`。
- Produces: 路由 `{ path: '/results', name: 'results', component: ResultsView }`。

- [ ] **Step 1: 寫失敗測試**

建立 `app/src/views/results/__tests__/resultsView.spec.js`：

```js
import { createLocalVue, mount } from '@vue/test-utils'
import Vuex from 'vuex'
import ResultsView from '../ResultsView.vue'
import query from '../../../store/modules/query'
import welfare from '../../../store/modules/welfare'
import eventService from '../../../services/eventService'

const localVue = createLocalVue()
localVue.use(Vuex)

const RouterLinkStub = {
  name: 'RouterLink',
  props: { to: { type: [String, Object], required: true } },
  render(h) { return h('a', { attrs: { href: typeof this.to === 'string' ? this.to : '#' } }, this.$slots.default) },
}

function render(q) {
  const store = new Vuex.Store({ modules: { query, welfare } })
  const w = mount(ResultsView, { localVue, store, stubs: { RouterLink: RouterLinkStub }, mocks: { $route: { query: { q } } } })
  return { w, store }
}

const flush = () => new Promise((resolve) => setTimeout(resolve, 0))

test('無法判斷 → 顯示提示與三個任務入口', async () => {
  const { w } = render('最近有什麼')
  await flush()
  expect(w.text()).toContain('換句話說')
  expect(w.find('a[href="/assistant/report"]').exists()).toBe(true)
  expect(w.find('a[href="/assistant/events"]').exists()).toBe(true)
  expect(w.find('a[href="/assistant/welfare"]').exists()).toBe(true)
})

test('複合 → 同時呈現事件與福利兩區', async () => {
  vi.spyOn(eventService, 'list').mockResolvedValue([])
  const { w } = render('附近有沒有積水，這個月有什麼老人活動')
  await flush()
  expect(w.text()).toContain('附近事件')
  expect(w.text()).toContain('福利與活動')
})

test('含回報意圖 → 顯示前往回報入口、不自動開始流程', async () => {
  vi.spyOn(eventService, 'list').mockResolvedValue([])
  const { w } = render('中華路有坑洞，這個月有什麼老人活動')
  await flush()
  const link = w.find('a[href="/assistant/report"]')
  expect(link.exists()).toBe(true)
  expect(link.text()).toContain('要回報嗎')
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/views/results/__tests__/resultsView.spec.js`
Expected: FAIL（找不到 `../ResultsView.vue`）

- [ ] **Step 3: 建立 `ResultsView.vue`**

- `created()`：`this.intents = parseIntents(this.$route.query.q || '')`；若含 `query` → `dispatch('query/searchFromText', clause)`；若含 `welfare` → `dispatch('welfare/search', clause)`。
- `computed`：`hasQuery`／`hasWelfare`／`hasReport`；事件區讀 `query` store（`events`／`loading`／`error`）；福利區讀 `welfare` store（`phase`／`results`／`partial`／`failedSources`／`timeout`／`busy`）。
- 模板：`h1` 頁標；`intents.length === 0` → 提示＋三個 `router-link`（`/assistant/report`、`/assistant/events`、`/assistant/welfare`）；否則分區（事件／福利／回報）。
- 回報區：`<router-link to="/assistant/report">要回報嗎？前往回報</router-link>`。
- 各區塊用 `h2`；載入用 `role="status"`。

- [ ] **Step 4: 調整路由並刪除釐清**

- `router/index.js`：新增 `/results` 路由；移除 `/clarify` 路由與 `ClarifyView` import。
- 刪除 `app/src/views/clarify/ClarifyView.vue` 與 `app/src/views/clarify/__tests__/clarifyView.spec.js`。

- [ ] **Step 5: 執行測試確認通過**

Run: `cd app; npx vitest run src/views/results/__tests__/resultsView.spec.js`
Expected: PASS

- [ ] **Step 6: 提交**

```bash
git add -A
git commit -m "feat(results): composite results view; remove clarify screen"
```

---

### Task 4: 全量驗證

**Files:** 無新增。

- [ ] **Step 1: 執行全部測試**

Run: `cd app; npm test`
Expected: 全部 PASS（含既有 `homeView`、`intent` 等）。

- [ ] **Step 2: 建置**

Run: `cd app; npm run build`
Expected: 成功產出 `dist/`。

- [ ] **Step 3: 人工複驗**

首頁輸入「附近有沒有積水，這個月有什麼老人活動」→ 進 `/results` 兩區都有內容；輸入「最近有什麼」→ 提示＋三入口；輸入「這個月有什麼老人活動」→ 直接進福利；確認全站找不到「你是想找哪一種」。

- [ ] **Step 4: 提交（若有微調）**

```bash
git add -A
git commit -m "chore: composite intent verification fixes"
```

---

## Self-Review

- **Spec coverage**：§4.1 → Task 1；§4.2 → Task 2；§4.3 → Task 3；§4.4 → Task 3；§4.5 → Global Constraints／各 Task。
- **Step scan**：每步單一可檢查動作；程式碼塊只給測試與簽名。
- **Type consistency**：`parseIntents` 回傳形狀、`routeFromText` 回傳 `{name, query}`、`ResultsView` 的 `intents`／`hasQuery`／`hasWelfare`／`hasReport` 在 Task 1–3 一致。
- **Review Focus**：五條皆指派（1→Task 3、2→Task 3、3→Task 2、4→Task 1、5→Task 3）。
- **Proportion**：計畫與規格相當。
