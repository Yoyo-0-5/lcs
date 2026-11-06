# 「我的回報」完整版 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把「我的回報」做完整——筆數、狀態篩選、進詳情、刪除（二次確認）、兩種空狀態；維持不持久化（記憶體）。

**Architecture:** 擴充 `myReports` store 的快照（加 `description`／`photo`）並新增 `remove`；`MyReportsView` 加入篩選與刪除（皆為頁面本地狀態）；新增 `MyReportDetailView` 由快照顯示詳情。

**Tech Stack:** Vue 2.7（Options API）、Vue Router 3、Vuex 3、Vite 5、Vitest 1 + @vue/test-utils 1。

**Spec:** `docs/superpowers/specs/2026-10-05-my-reports-full-design.md`

## Global Constraints

- **不持久化**：維持 `myReports` 記憶體 store，重整即清空。
- 詳情**由快照顯示**，不查事件來源。
- 篩選與刪除確認皆為**頁面本地狀態**，不放 store。
- 刪除需**二次確認**才真的移除。
- 狀態用語一致：`reported→待處理`、`in-progress→處理中`、`resolved→已完成`。
- 畫面不得出現 Agent／AI／API 等技術詞（G5）。
- 沿用入口基準 token；≥48px、WCAG AA、可見焦點、`prefers-reduced-motion`。
- 測試指令：`cd app; npm test`。

## Review Focus

1. **空清單 → 空狀態；有紀錄但篩選後為空 → 篩選空狀態**（兩種分明）。（Task 2）
2. **刪除兩步**：未按「確定刪除？」不得移除；可按「取消」。（Task 2）
3. **詳情找不到 id** → 顯示「這筆回報已移除」＋回清單，不崩潰。（Task 3）
4. **快照含 `description`／`photo`；`remove` 正確刪除對應 id**。（Task 1）
5. **筆數**：有篩選且少於總數時顯示「共 N 筆（全部 M 筆）」。（Task 2）

---

### Task 1: myReports store 擴充與刪除

**Files:**
- Modify: `app/src/store/modules/myReports.js`
- Modify: `app/src/store/__tests__/myReports.spec.js`

**Interfaces:**
- Produces:
  - 快照：`{ id, title, type, placeText, timeText, status, reportedAt, description, photo }`
  - mutation：`myReports/remove(state, id)`（由 `list` 移除該 id）

- [ ] **Step 1: 寫失敗測試**

在 `app/src/store/__tests__/myReports.spec.js` 追加：

```js
const EVENT_WITH_DETAIL = {
  id: 'e-200', title: '路燈不亮', type: '路燈', placeText: '汀州路', timeText: '昨晚',
  status: 'reported', reportedAt: '2026-10-05T20:00:00+08:00',
  description: '整排路燈不亮', photo: 'data:image/png;base64,AAA',
}

test('add 會保留 description 與 photo', () => {
  const store = new Vuex.Store({ modules: { myReports } })
  store.commit('myReports/add', EVENT_WITH_DETAIL)
  expect(store.state.myReports.list[0].description).toBe('整排路燈不亮')
  expect(store.state.myReports.list[0].photo).toBe('data:image/png;base64,AAA')
})

test('remove 會移除對應 id', () => {
  const store = new Vuex.Store({ modules: { myReports } })
  store.commit('myReports/add', EVENT_WITH_DETAIL)
  store.commit('myReports/remove', 'e-200')
  expect(store.state.myReports.list).toHaveLength(0)
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/store/__tests__/myReports.spec.js`
Expected: FAIL（`description` 未保留／`remove` 未定義）

- [ ] **Step 3: 修改 `myReports.js`**

- `add` 的快照加入 `description: event.description || ''`、`photo: event.photo || null`。
- 新增 mutation `remove(state, id) { state.list = state.list.filter((item) => item.id !== id) }`。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/store/__tests__/myReports.spec.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add app/src/store/modules/myReports.js app/src/store/__tests__/myReports.spec.js
git commit -m "feat(my-reports): snapshot detail fields and remove mutation"
```

---

### Task 2: 清單頁（筆數、篩選、刪除）

**Files:**
- Modify: `app/src/views/my/MyReportsView.vue`
- Modify: `app/src/views/my/__tests__/myReportsView.spec.js`

**Interfaces:**
- Consumes: `myReports`（Task 1）、`ResultCard`、`EmptyState`。
- Produces: 本地狀態 `statusFilter`（`''`＝全部）與 `pendingDeleteId`。

- [ ] **Step 1: 寫失敗測試**

改寫 `app/src/views/my/__tests__/myReportsView.spec.js` 的 `mountView`（加 `RouterLinkStub`）並追加：

```js
const RouterLinkStub = {
  name: 'RouterLink',
  props: { to: { type: [String, Object], required: true } },
  render(h) { return h('a', { attrs: { href: typeof this.to === 'string' ? this.to : '#' } }, this.$slots.default) },
}
function mountView(store) {
  return mount(MyReportsView, { localVue, store, stubs: { RouterLink: RouterLinkStub } })
}
function seed(store) {
  store.commit('myReports/add', { id: 'e-1', title: 'A', status: 'reported', type: '坑洞' })
  store.commit('myReports/add', { id: 'e-2', title: 'B', status: 'resolved', type: '路燈' })
}

test('顯示筆數', () => {
  const store = new Vuex.Store({ modules: { myReports } })
  seed(store)
  const w = mountView(store)
  expect(w.text()).toContain('共 2 筆')
})

test('狀態篩選只顯示符合者，筆數顯示全部', async () => {
  const store = new Vuex.Store({ modules: { myReports } })
  seed(store)
  const w = mountView(store)
  const chip = w.findAll('.my-reports__chip').wrappers.find((b) => b.text() === '已完成')
  await chip.trigger('click')
  expect(w.findAllComponents({ name: 'ResultCard' })).toHaveLength(1)
  expect(w.text()).toContain('共 1 筆（全部 2 筆）')
})

test('刪除需二次確認才移除', async () => {
  const store = new Vuex.Store({ modules: { myReports } })
  seed(store)
  const w = mountView(store)
  await w.findAll('.my-reports__delete').at(0).trigger('click')
  expect(store.state.myReports.list).toHaveLength(2) // 尚未刪除
  const confirm = w.findAll('.my-reports__delete').at(0)
  expect(confirm.text()).toContain('確定刪除')
  await confirm.trigger('click')
  expect(store.state.myReports.list).toHaveLength(1)
})

test('完全沒有紀錄時的空狀態', () => {
  const store = new Vuex.Store({ modules: { myReports } })
  const w = mountView(store)
  expect(w.text()).toContain('還沒有回報紀錄')
})

test('篩選後為空的訊息（與完全沒有區分）', async () => {
  const store = new Vuex.Store({ modules: { myReports } })
  store.commit('myReports/add', { id: 'e-1', title: 'A', status: 'reported' })
  const w = mountView(store)
  const chip = w.findAll('.my-reports__chip').wrappers.find((b) => b.text() === '已完成')
  await chip.trigger('click')
  expect(w.text()).toContain('這個狀態還沒有回報')
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/views/my/__tests__/myReportsView.spec.js`
Expected: FAIL（無 `.my-reports__chip`／`.my-reports__delete`／筆數）

- [ ] **Step 3: 改寫 `MyReportsView.vue`**

- `data`：`statusFilter: ''`、`pendingDeleteId: null`；`statusOptions`（`''`／`reported`／`in-progress`／`resolved` 對應 全部／待處理／處理中／已完成）。
- `computed`：
  - `list`（store）、`filtered`（`statusFilter` 為空則全部，否則比對 `status`）、`countText`。
- `countText`：`statusFilter` 有值且 `filtered.length !== list.length` → `共 ${filtered.length} 筆（全部 ${list.length} 筆）`；否則 `共 ${list.length} 筆`。
- 模板：
  - `h1`「我的回報」＋筆數。
  - 篩選：`.my-reports__chip`（`role="group"`、`aria-pressed`）→ 設定 `statusFilter`。
  - `list.length === 0` → `EmptyState`「還沒有回報紀錄」＋前往回報（`/assistant/report`）。
  - `list.length > 0 && filtered.length === 0` → 「這個狀態還沒有回報」＋「看全部」按鈕（`statusFilter = ''`）。
  - 否則：每筆 `ResultCard`；動作區：
    - `.my-reports__view` → `<router-link :to="{ name: 'my-report-detail', params: { id: item.id } }">查看</router-link>`。
    - `pendingDeleteId === item.id` → `.my-reports__delete`「確定刪除？」（`confirmDelete`）＋「取消」（`cancelDelete`）。
    - 否則 → `.my-reports__delete`「刪除」（`askDelete(item.id)`）。
- `methods`：`askDelete(id)`、`confirmDelete(id)`（`dispatch('myReports/remove', id)` 並清 `pendingDeleteId`）、`cancelDelete()`。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/views/my/__tests__/myReportsView.spec.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add app/src/views/my/MyReportsView.vue app/src/views/my/__tests__/myReportsView.spec.js
git commit -m "feat(my-reports): count, status filter, delete with confirm"
```

---

### Task 3: 詳情頁

**Files:**
- Create: `app/src/views/my/MyReportDetailView.vue`
- Create: `app/src/views/my/__tests__/myReportDetailView.spec.js`
- Modify: `app/src/router/index.js`（新增 `/my-reports/:id`）

**Interfaces:**
- Consumes: `myReports`（Task 1）。
- Produces: 路由 `{ path: '/my-reports/:id', name: 'my-report-detail', component: MyReportDetailView }`。

- [ ] **Step 1: 寫失敗測試**

建立 `app/src/views/my/__tests__/myReportDetailView.spec.js`：

```js
import { createLocalVue, mount } from '@vue/test-utils'
import Vuex from 'vuex'
import MyReportDetailView from '../MyReportDetailView.vue'
import myReports from '../../../store/modules/myReports'

const localVue = createLocalVue()
localVue.use(Vuex)

const RouterLinkStub = {
  name: 'RouterLink',
  props: { to: { type: [String, Object], required: true } },
  render(h) { return h('a', { attrs: { href: typeof this.to === 'string' ? this.to : '#' } }, this.$slots.default) },
}

function render(id) {
  const store = new Vuex.Store({ modules: { myReports } })
  store.commit('myReports/add', {
    id: 'e-1', title: '路燈不亮', type: '路燈', placeText: '汀州路', timeText: '昨晚',
    status: 'reported', description: '整排路燈不亮', photo: 'data:image/png;base64,AAA',
  })
  const w = mount(MyReportDetailView, {
    localVue, store, stubs: { RouterLink: RouterLinkStub }, mocks: { $route: { params: { id } } },
  })
  return { w, store }
}

test('顯示快照的標題、描述與照片', () => {
  const { w } = render('e-1')
  expect(w.text()).toContain('路燈不亮')
  expect(w.text()).toContain('整排路燈不亮')
  expect(w.find('img').attributes('src')).toBe('data:image/png;base64,AAA')
})

test('找不到 id → 顯示已移除與回清單連結', () => {
  const { w } = render('nope')
  expect(w.text()).toContain('這筆回報已移除')
  expect(w.find('a[href="/my-reports"]').exists()).toBe(true)
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/views/my/__tests__/myReportDetailView.spec.js`
Expected: FAIL（找不到 `../MyReportDetailView.vue`）

- [ ] **Step 3: 建立 `MyReportDetailView.vue` 並加路由**

- `computed.report`：`this.$store.state.myReports.list.find((r) => r.id === this.$route.params.id) || null`。
- 找不到 → `EmptyState`「這筆回報已移除」，action 為 `<router-link to="/my-reports">回我的回報</router-link>`。
- 找到 → `h1` 標題；`dl` 顯示類型／狀態／地點／時間；描述（無則「沒有補充描述。」）；照片（`img`，無則「這則回報沒有附照片」）；`<router-link to="/my-reports">回我的回報</router-link>`。
- 狀態文字同既有映射。
- `router/index.js`：新增 `/my-reports/:id` 路由（import 元件）。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/views/my/__tests__/myReportDetailView.spec.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add app/src/views/my/MyReportDetailView.vue app/src/views/my/__tests__/myReportDetailView.spec.js app/src/router/index.js
git commit -m "feat(my-reports): detail view from snapshot"
```

---

### Task 4: 全量驗證

**Files:** 無新增。

- [ ] **Step 1: 執行全部測試**

Run: `cd app; npm test`
Expected: 全部 PASS。

- [ ] **Step 2: 建置**

Run: `cd app; npm run build`
Expected: 成功產出 `dist/`。

- [ ] **Step 3: 人工複驗**

送出一筆回報 → `/my-reports` 出現該筆且筆數正確 → 篩選狀態 → 查看詳情（描述／照片）→ 刪除（二次確認）→ 篩選後為空顯示對應訊息。

- [ ] **Step 4: 提交（若有微調）**

```bash
git add -A
git commit -m "chore: my reports full verification fixes"
```

---

## Self-Review

- **Spec coverage**：§4.1 → Task 1；§4.2 → Task 2；§4.3 → Task 3；§4.4 → Global Constraints。
- **Step scan**：每步單一可檢查動作；程式碼塊只給測試與簽名。
- **Type consistency**：快照欄位、`myReports/remove`、`statusFilter`／`pendingDeleteId`、路由 name `my-report-detail` 在 Task 1–3 一致。
- **Review Focus**：五條皆指派（1→Task 2、2→Task 2、3→Task 3、4→Task 1、5→Task 2）。
- **Proportion**：計畫與規格相當。
