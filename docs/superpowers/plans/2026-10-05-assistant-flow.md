# 助手對話入口＋導覽調整 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 首頁三張卡與手機底部導覽改為「點擊即開始該任務的引導式對話」，問完接手既有流程；桌面頂部導覽改放「事件總覽」與「我的回報」，並提供一個最小可用的「我的回報」頁。

**Architecture:** 以 `services/assistantScripts.js`（純資料 + 純 `finish`）描述每個任務的一問一答步驟；`AssistantView` 依腳本渲染並在最後套用 `finish` 回傳的 commits／route。導覽資料集中於 `router/index.js`。`myReports` 記憶體 store 記錄本工作階段送出的回報，`MyReportsView` 呈現。

**Tech Stack:** Vue 2.7（Options API）、Vue Router 3、Vuex 3、Vite 5、Vitest 1 + @vue/test-utils 1。

**Spec:** `docs/superpowers/specs/2026-10-05-assistant-flow-design.md`

## Global Constraints

- 引導式一問一答：一次只問一題；選項題為大按鈕、文字題可打字。
- 畫面**不得出現 Agent／AI／API 等技術詞**（G5）；用白話。
- 既有流程（回報 3 步、事件清單、福利搜尋）行為不變，只把答案寫進 store 再導頁。
- 交接：回報 → `report/setDescription` → `/report/location`；事件 → `query/setFilters`／`query/setRegion` → `/events`；福利 → `/welfare?q=`。
- 可點元素 ≥ 48px、WCAG AA、可見焦點、`prefers-reduced-motion`；沿用入口基準 token。
- `finish(answers)` 為純函式，不直接碰 store／router。
- 「我的回報」本版僅**記憶體**（本工作階段），不做持久化。
- 測試指令：於 `app/` 執行 `npm test`。

## Review Focus

1. **無效 `:task` 不得崩潰**：`/assistant/unknown` 應導回首頁。（Task 3）
2. **文字題留空不得前進**。（Task 3）
3. **事件對話正確寫入篩選**：`time` 一定寫入；「都可以」不寫 `region`。（Task 2）
4. **回報送出後 `myReports` 新增、同 id 不重複列**。（Task 1）
5. **手機底部導覽改指向對話後，選中態仍正確**（在 `/` 時「首頁」亮、其他不誤亮）。（Task 5）

---

### Task 1: myReports 記憶體 store 與送出掛接

**Files:**
- Create: `app/src/store/modules/myReports.js`
- Modify: `app/src/store/index.js`
- Modify: `app/src/store/modules/report.js`
- Test: `app/src/store/__tests__/myReports.spec.js`

**Interfaces:**
- Produces:
  - `myReports` 模組：`state.list: Array<Snapshot>`；mutation `myReports/add(event)`。
  - Snapshot：`{ id, title, type, placeText, timeText, status, reportedAt }`（缺漏以空字串／`'reported'` 補）。
  - `report/submit` 成功後會 `commit('myReports/add', event, { root: true })`。

- [ ] **Step 1: 寫失敗測試**

建立 `app/src/store/__tests__/myReports.spec.js`：

```js
import { createLocalVue } from '@vue/test-utils'
import Vuex from 'vuex'
import myReports from '../modules/myReports'
import report from '../modules/report'

const localVue = createLocalVue()
localVue.use(Vuex)

const EVENT = {
  id: 'e-100', title: '中華路坑洞', type: '坑洞', placeText: '中華路一段',
  timeText: '剛剛', status: 'reported', reportedAt: '2026-10-05T10:00:00+08:00',
}

test('add 會新增一筆快照（最新的在前）', () => {
  const store = new Vuex.Store({ modules: { myReports } })
  store.commit('myReports/add', EVENT)
  expect(store.state.myReports.list).toHaveLength(1)
  expect(store.state.myReports.list[0]).toMatchObject({ id: 'e-100', title: '中華路坑洞' })
})

test('同 id 重複 add 不重複列', () => {
  const store = new Vuex.Store({ modules: { myReports } })
  store.commit('myReports/add', EVENT)
  store.commit('myReports/add', EVENT)
  expect(store.state.myReports.list).toHaveLength(1)
})

test('report/submit 成功後會寫入 myReports', async () => {
  const store = new Vuex.Store({ modules: { myReports, report } })
  store.commit('report/setDescription', '中華路坑洞')
  store.commit('report/setLocation', { text: '中華路一段' })
  await store.dispatch('report/submit')
  expect(store.state.myReports.list).toHaveLength(1)
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/store/__tests__/myReports.spec.js`
Expected: FAIL（找不到 `../modules/myReports`）

- [ ] **Step 3: 建立 `app/src/store/modules/myReports.js`**

```js
// 本工作階段送出的回報（記憶體）。跨重整的持久化屬第二份規格。
export default {
  namespaced: true,
  state: () => ({ list: [] }),
  mutations: {
    add(state, event) {
      if (!event || event.id == null) return
      const snapshot = {
        id: event.id,
        title: event.title || '',
        type: event.type || '',
        placeText: event.placeText || '',
        timeText: event.timeText || '',
        status: event.status || 'reported',
        reportedAt: event.reportedAt || '',
      }
      state.list = [snapshot, ...state.list.filter((item) => item.id !== snapshot.id)]
    },
  },
}
```

- [ ] **Step 4: 註冊模組並在送出時掛接**

- `app/src/store/index.js`：import `myReports` 並加入 `modules`。
- `app/src/store/modules/report.js` 的 `submit`：取得 `event` 後 `commit('myReports/add', event, { root: true })`（在 `setSubmittedId` 之後、`setStatus('submitted')` 之前）。

- [ ] **Step 5: 執行測試確認通過**

Run: `cd app; npx vitest run src/store/__tests__/myReports.spec.js`
Expected: PASS

- [ ] **Step 6: 提交**

```bash
git add app/src/store/modules/myReports.js app/src/store/index.js app/src/store/modules/report.js app/src/store/__tests__/myReports.spec.js
git commit -m "feat(my-reports): in-session myReports store wired to report submit"
```

---

### Task 2: 任務腳本 assistantScripts

**Files:**
- Create: `app/src/services/assistantScripts.js`
- Test: `app/src/services/__tests__/assistantScripts.spec.js`

**Interfaces:**
- Produces: `ASSISTANT_SCRIPTS`（`{ [task]: { title, intro, steps, finish } }`）與具名 `getScript(task)`。
  - `step`：`{ id, type: 'text'|'choice', question, placeholder?, options? }`；`choice` 的 option 為 `{ value, label }`。
  - `finish(answers)` 回傳 `{ commits: Array<{type,payload}>, route: object }`。

- [ ] **Step 1: 寫失敗測試**

建立 `app/src/services/__tests__/assistantScripts.spec.js`：

```js
import { ASSISTANT_SCRIPTS, getScript } from '../assistantScripts'

test('三個任務都有腳本', () => {
  expect(Object.keys(ASSISTANT_SCRIPTS).sort()).toEqual(['events', 'report', 'welfare'])
  expect(getScript('nope')).toBeNull()
})

test('report.finish 寫入描述並轉往選位置', () => {
  const out = ASSISTANT_SCRIPTS.report.finish({ description: '中華路坑洞' })
  expect(out.commits).toEqual([{ type: 'report/setDescription', payload: '中華路坑洞' }])
  expect(out.route).toEqual({ name: 'report-location' })
})

test('events.finish 寫入時間；有地區才寫 region', () => {
  const out = ASSISTANT_SCRIPTS.events.finish({ time: 'week', region: '中華路' })
  expect(out.commits).toContainEqual({ type: 'query/setFilters', payload: { time: 'week' } })
  expect(out.commits).toContainEqual({ type: 'query/setRegion', payload: '中華路' })
  expect(out.route).toEqual({ name: 'events' })

  const none = ASSISTANT_SCRIPTS.events.finish({ time: 'month', region: '' })
  expect(none.commits).toEqual([{ type: 'query/setFilters', payload: { time: 'month' } }])
})

test('welfare.finish 以非空答案組成 ?q=', () => {
  expect(ASSISTANT_SCRIPTS.welfare.finish({ type: '補助', region: '汀州路' }).route)
    .toEqual({ name: 'welfare', query: { q: '補助 汀州路' } })
  expect(ASSISTANT_SCRIPTS.welfare.finish({ type: '活動', region: '' }).route)
    .toEqual({ name: 'welfare', query: { q: '活動' } })
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/services/__tests__/assistantScripts.spec.js`
Expected: FAIL（找不到模組）

- [ ] **Step 3: 建立 `app/src/services/assistantScripts.js`**

依 Interfaces 實作三個腳本（intro／steps／finish）。`report` 的 `description` 步驟為 `text`（必填）。`events`／`welfare` 的地區選項需與種子資料一致（中華路／汀州路／青年公園／台北車站，最後一個為「都可以」`value: ''`）。`getScript(task)` 回傳 `ASSISTANT_SCRIPTS[task] || null`。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/services/__tests__/assistantScripts.spec.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add app/src/services/assistantScripts.js app/src/services/__tests__/assistantScripts.spec.js
git commit -m "feat(assistant): task scripts with pure finish handoff"
```

---

### Task 3: AssistantView 與對話路由

**Files:**
- Create: `app/src/views/assistant/AssistantView.vue`
- Modify: `app/src/router/index.js`（新增 `/assistant/:task`）
- Modify: `app/src/views/home/HomeView.vue`（三張卡改連對話頁）
- Test: `app/src/views/assistant/__tests__/assistantView.spec.js`

**Interfaces:**
- Consumes: `getScript`（Task 2）；`report/setPhoto`／`setAudio`（既有）。
- Produces: 路由 `{ path: '/assistant/:task', name: 'assistant', component: AssistantView }`。

- [ ] **Step 1: 寫失敗測試**

建立 `app/src/views/assistant/__tests__/assistantView.spec.js`：

```js
import { createLocalVue, mount } from '@vue/test-utils'
import Vuex from 'vuex'
import AssistantView from '../AssistantView.vue'
import report from '../../../store/modules/report'
import myReports from '../../../store/modules/myReports'

const localVue = createLocalVue()
localVue.use(Vuex)

function render(task) {
  const store = new Vuex.Store({ modules: { report, myReports } })
  const $router = { push: vi.fn(), replace: vi.fn() }
  const $route = { params: { task } }
  const w = mount(AssistantView, {
    localVue,
    store,
    mocks: { $route, $router },
    stubs: ['AiPromptBar', 'router-link'],
  })
  return { w, store, $router }
}

test('顯示第一個問題與進度', () => {
  const { w } = render('events')
  expect(w.text()).toContain('想找多久以內？')
  expect(w.text()).toContain('第 1 題')
})

test('選項題選完前進到下一題', async () => {
  const { w } = render('events')
  await w.findAll('[data-choice]').at(0).trigger('click')
  expect(w.text()).toContain('哪一區？')
})

test('文字題留空不前進', async () => {
  const { w, $router } = render('report')
  w.findComponent({ name: 'AiPromptBar' }).vm.$emit('submit')
  expect($router.push).not.toHaveBeenCalled()
})

test('完成後套用 commits 並導頁', async () => {
  const { w, store, $router } = render('events')
  await w.findAll('[data-choice]').at(0).trigger('click') // time
  await w.findAll('[data-choice]').at(0).trigger('click') // region
  expect(store.state.query.filters.time).toBe('today')
  expect($router.push).toHaveBeenCalledWith({ name: 'events' })
})

test('無效的 task 導回首頁', () => {
  const { $router } = render('unknown')
  expect($router.replace).toHaveBeenCalledWith('/')
})
```

（`AiPromptBar` 以 stub 取代，避免相依語音／附件；report 的文字題另於 Task 內以 `setDescription` 驗證。）

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/views/assistant/__tests__/assistantView.spec.js`
Expected: FAIL（找不到 `../AssistantView.vue`）

- [ ] **Step 3: 建立 `AssistantView.vue`**

- `created()`：`this.script = getScript(this.$route.params.task)`；若 `null` → `this.$router.replace('/')`。
- `data`：`script`、`stepIndex: 0`、`answers: {}`、`text: ''`。
- `computed`：`currentStep = script && script.steps[stepIndex]`；`progress`（`第 N 題 / 共 M 題`）。
- 模板：
  - 標題用 `script.title`；開場白用 `script.intro`。
  - `choice` 步驟：對每個 option 渲染按鈕 `[data-choice]`（≥48px），`@click="answer(option.value)"`。
  - `text` 步驟：渲染 `AiPromptBar`（`v-model="text"`，`@submit="answer(text)"`）；附件事件 `@attach-photo`／`@attach-audio` 以 `report/setPhoto`／`setAudio` 寫入（同 `DescribeStep`）。
  - 「回首頁」按鈕 → `$router.push('/')`。
  - 問題元素加 `ref="question"`、`tabindex="-1"`。
- `answer(value)`：文字題若 `String(value).trim()===''` 直接 return；`this.answers[currentStep.id]=value`；若最後一步 → `finish()`，否則 `stepIndex++` 並在 `$nextTick` 對 `this.$refs.question.focus()`。
- `finish()`：`const { commits, route } = this.script.finish(this.answers)`；`commits.forEach(c => this.$store.commit(c.type, c.payload))`；`this.$router.push(route)`。

- [ ] **Step 4: 修改路由與首頁卡片**

- `app/src/router/index.js`：新增 `/assistant/:task` 路由（import `AssistantView`）。
- `app/src/views/home/HomeView.vue`：三張 `BigTaskCard` 的 `to` 改為 `/assistant/report`、`/assistant/events`、`/assistant/welfare`。

- [ ] **Step 5: 執行測試確認通過**

Run: `cd app; npx vitest run src/views/assistant/__tests__/assistantView.spec.js src/views/home/__tests__/homeView.spec.js`
Expected: PASS（home 既有測試仍綠）

- [ ] **Step 6: 提交**

```bash
git add app/src/views/assistant/AssistantView.vue app/src/router/index.js app/src/views/home/HomeView.vue app/src/views/assistant/__tests__/assistantView.spec.js
git commit -m "feat(assistant): guided conversation view and home card entry"
```

---

### Task 4: MyReportsView 與路由

**Files:**
- Create: `app/src/views/my/MyReportsView.vue`
- Modify: `app/src/router/index.js`（新增 `/my-reports`）
- Test: `app/src/views/my/__tests__/myReportsView.spec.js`

**Interfaces:**
- Consumes: `myReports`（Task 1）、`ResultCard`、`EmptyState`。
- Produces: 路由 `{ path: '/my-reports', name: 'my-reports', component: MyReportsView }`。

- [ ] **Step 1: 寫失敗測試**

建立 `app/src/views/my/__tests__/myReportsView.spec.js`：

```js
import { createLocalVue, mount } from '@vue/test-utils'
import Vuex from 'vuex'
import MyReportsView from '../MyReportsView.vue'
import myReports from '../../../store/modules/myReports'

const localVue = createLocalVue()
localVue.use(Vuex)

const RouterLinkStub = {
  name: 'RouterLink',
  props: { to: { type: [String, Object], required: true } },
  render(h) { return h('a', this.$slots.default) },
}

function mountView(store) {
  return mount(MyReportsView, { localVue, store, stubs: { RouterLink: RouterLinkStub } })
}

test('沒有紀錄時顯示空狀態', () => {
  const store = new Vuex.Store({ modules: { myReports } })
  const w = mountView(store)
  expect(w.text()).toContain('還沒有回報紀錄')
})

test('有紀錄時以卡片列出', () => {
  const store = new Vuex.Store({ modules: { myReports } })
  store.commit('myReports/add', {
    id: 'e-100', title: '中華路坑洞', placeText: '中華路一段', timeText: '剛剛', status: 'reported',
  })
  const w = mountView(store)
  expect(w.findAllComponents({ name: 'ResultCard' })).toHaveLength(1)
  expect(w.text()).toContain('中華路坑洞')
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/views/my/__tests__/myReportsView.spec.js`
Expected: FAIL（找不到 `../MyReportsView.vue`）

- [ ] **Step 3: 建立 `MyReportsView.vue`**

- `computed.list = this.$store.state.myReports.list`。
- 空：`<EmptyState message="還沒有回報紀錄">`，action 為 `<router-link to="/assistant/report">回報一個問題</router-link>`。
- 非空：`<ul>` 列出 `<ResultCard :title :date-text="item.timeText" :place-text="item.placeText" :badge="statusText(item.status)" />`；`statusText` 對應 `reported→待處理`、`in-progress→處理中`、`resolved→已完成`。
- 標題 `h1`「我的回報」；沿用入口基準 token。

- [ ] **Step 4: 新增路由**

`app/src/router/index.js`：新增 `/my-reports` 路由（import `MyReportsView`）。

- [ ] **Step 5: 執行測試確認通過**

Run: `cd app; npx vitest run src/views/my/__tests__/myReportsView.spec.js`
Expected: PASS

- [ ] **Step 6: 提交**

```bash
git add app/src/views/my/MyReportsView.vue app/src/router/index.js app/src/views/my/__tests__/myReportsView.spec.js
git commit -m "feat(my-reports): minimal in-session my reports page"
```

---

### Task 5: 導覽調整（桌面頂部、手機底部）

**Files:**
- Modify: `app/src/router/index.js`（`NAV_ITEMS` 的 `to` 改為對話頁；新增 `DESKTOP_NAV_ITEMS`）
- Modify: `app/src/components/TopNav.vue`（移除三項，改放事件總覽／我的回報）
- Test: `app/src/components/__tests__/nav.spec.js`

**Interfaces:**
- Consumes: 路由 `/assistant/:task`、`/events`、`/my-reports`。
- Produces:
  - `NAV_ITEMS`：三項，`to` 改為 `/assistant/report`、`/assistant/events`、`/assistant/welfare`（label／icon 不變）。
  - `MOBILE_NAV_ITEMS = [{ to:'/', label:'首頁', icon:'House' }, ...NAV_ITEMS]`（不變的組合方式）。
  - `DESKTOP_NAV_ITEMS = [{ to:'/events', label:'事件總覽', icon:'MapPin' }, { to:'/my-reports', label:'我的回報', icon:'Flag' }]`。

- [ ] **Step 1: 寫失敗測試**

改寫 `nav.spec.js` 中「桌面導覽」相關測試為：

```js
test('手機底部導覽三項指向對話頁、含首頁', () => {
  const w = mount(BottomNav, { stubs: ['router-link'] })
  const items = w.findAll('[data-nav-item]')
  expect(items).toHaveLength(4)
  expect(w.text()).toContain('首頁')
})

test('NAV_ITEMS 指向對話頁', () => {
  expect(NAV_ITEMS.map((i) => i.to)).toEqual([
    '/assistant/report', '/assistant/events', '/assistant/welfare',
  ])
})

test('桌面導覽顯示事件總覽與我的回報', () => {
  const w = mount(TopNav, { stubs: ['router-link'] })
  const items = w.findAll('[data-nav-item]')
  expect(items).toHaveLength(2)
  expect(w.text()).toContain('事件總覽')
  expect(w.text()).toContain('我的回報')
  expect(w.text()).not.toContain('福利活動')
})
```

（import `DESKTOP_NAV_ITEMS` 一併驗證其 `to`。）

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/components/__tests__/nav.spec.js`
Expected: FAIL（TopNav 仍顯示三項且無「我的回報」）

- [ ] **Step 3: 調整導覽資料與 TopNav**

- `router/index.js`：改 `NAV_ITEMS` 的 `to`；新增 `DESKTOP_NAV_ITEMS`。
- `TopNav.vue`：移除 `NAV_ITEMS` 清單渲染；改渲染品牌 + `DESKTOP_NAV_ITEMS`（沿用既有 pill 樣式與 `iconFor`）。
- `BottomNav.vue`：不需改（其資料來自 `MOBILE_NAV_ITEMS`，`to` 已隨 `NAV_ITEMS` 更新）。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/components/__tests__/nav.spec.js src/components/__tests__/appShell.spec.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add app/src/router/index.js app/src/components/TopNav.vue app/src/components/__tests__/nav.spec.js
git commit -m "feat(nav): desktop header shows events overview and my reports; mobile nav opens conversations"
```

---

### Task 6: 全量驗證

**Files:** 無新增。

- [ ] **Step 1: 執行全部測試**

Run: `cd app; npm test`
Expected: 全部 PASS。

- [ ] **Step 2: 建置**

Run: `cd app; npm run build`
Expected: 成功產出 `dist/`。

- [ ] **Step 3: 人工複驗**

於 375px / 1024px 檢查：點卡片進對話並可完成、接手正確流程；桌面 header 顯示「事件總覽」「我的回報」；手機底部三項觸發對話；送出回報後 `/my-reports` 出現該筆；無效 `/assistant/xxx` 回首頁。

- [ ] **Step 4: 提交（若有微調）**

```bash
git add -A
git commit -m "chore: assistant flow verification fixes"
```

---

## Self-Review

- **Spec coverage**：§2.1 對話頁/TopNav/BottomNav/我的回報最小版 → Task 2/3/4/5；§4.3 腳本 → Task 2；§4.7 導覽 → Task 5；§4.8 我的回報 → Task 1/4。
- **Step scan**：每步單一可檢查動作；程式碼塊只給測試與簽名。
- **Type consistency**：`ASSISTANT_SCRIPTS`／`getScript`、`step.type`、`finish→{commits,route}`、`myReports/add`、`NAV_ITEMS`／`DESKTOP_NAV_ITEMS` 在 Task 1–5 一致。
- **Review Focus**：五條皆指派（1→Task 3、2→Task 3、3→Task 2、4→Task 1、5→Task 5）。
- **Proportion**：計畫與規格相當。
