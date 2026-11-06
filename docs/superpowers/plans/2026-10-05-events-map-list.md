# 附近事件：地圖＋清單 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在「附近事件」頁加入唯讀多圖釘的 Google 地圖，與事件清單雙向連動，並在無金鑰時優雅降級為純清單。

**Architecture:** 先把 `MapPicker` 內部的 Google Maps 載入器抽成共用模組 `services/googleMaps.js`；新增唯讀多圖釘元件 `EventMap.vue` 消費該載入器；`EventListView.vue` 改為「地圖＋清單」版面，以編號讓圖釘與卡片對應。無金鑰或載入失敗時，地圖顯示提示、清單照常。

**Tech Stack:** Vue 2.7（Options API）、Vuex 3、Vite 5、Vitest 1 + @vue/test-utils 1、Google Maps JavaScript API（`VITE_GOOGLE_MAPS_KEY`）。

**Spec:** `docs/superpowers/specs/2026-10-05-events-map-list-design.md`

## Global Constraints

- 地圖來源為 Google Maps；金鑰取自 `import.meta.env.VITE_GOOGLE_MAPS_KEY`。
- **測試環境與無金鑰時不得注入真實 API**：`loadGoogleMaps()` 在無 `window.google` 且無金鑰時回 `null`，不新增 `<script>`。
- **不合成座標**：圖釘只使用事件資料既有的 `coordinates`；`coordinates` 為 `null` 者不放圖釘。
- **清單是主要介面**：v1 不要求操作地圖；讀屏／鍵盤使用者只靠清單即可取得全部資訊。
- 沿用入口基準的 token；可點元素 ≥48px、WCAG AA 對比、可見焦點、尊重 `prefers-reduced-motion`。
- 測試指令：於 `app/` 執行 `npm test`（=`vitest run`）。

## Review Focus

以下五類是規格隱含、單元測試不易涵蓋、最可能在使用者手上出錯的情況；每條都在對應任務加上測試：

1. **無金鑰／載入失敗時不得丟例外**，且清單仍完整可用。（Task 2）
2. **座標為 `null` 時不得放圖釘、不得合成座標**。（Task 2）
3. **點圖釘要 emit 正確的事件 id**（編號不得與卡片錯位）。（Task 2）
4. **`events` 變更（例如切換篩選）時要重建圖釘**，不殘留舊 marker。（Task 2）
5. **無座標事件要列於清單最後並標「位置未標示」**，且不影響其餘清單閱讀。（Task 3）

---

### Task 1: 抽出共用 Google Maps 載入模組

**Files:**
- Create: `app/src/services/googleMaps.js`
- Modify: `app/src/components/MapPicker.vue`（移除內部 `KEY`／`loadGoogleMaps`，改用共用模組）
- Test: `app/src/services/__tests__/googleMaps.spec.js`

**Interfaces:**
- Produces: `loadGoogleMaps(): Promise<object|null>` — 回傳 `window.google.maps`，或在無 `window.google` 且無金鑰時回 `null`。

- [ ] **Step 1: 寫失敗測試**

建立 `app/src/services/__tests__/googleMaps.spec.js`：

```js
import { loadGoogleMaps } from '../googleMaps'

afterEach(() => {
  delete window.google
})

test('沒有金鑰且無 window.google 時回 null（不注入真實 API）', async () => {
  await expect(loadGoogleMaps()).resolves.toBeNull()
})

test('window.google 已存在時直接回傳，不重複載入', async () => {
  const maps = { Map: function () {}, Marker: function () {} }
  window.google = { maps }
  await expect(loadGoogleMaps()).resolves.toBe(maps)
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/services/__tests__/googleMaps.spec.js`
Expected: FAIL（找不到模組 `../googleMaps`）

- [ ] **Step 3: 建立 `app/src/services/googleMaps.js`**

把 `MapPicker.vue` 現有的 `KEY`、`mapsOrNull`、`loadGoogleMaps` 原封不動搬過來，並 `export function loadGoogleMaps()`（保留「無金鑰／測試環境回 null」的行為）。

- [ ] **Step 4: 修改 `MapPicker.vue` 改用共用模組**

刪除檔內的 `KEY`／`mapsOrNull`／`loadGoogleMaps`，改為 `import { loadGoogleMaps } from '../services/googleMaps'`。其餘行為不變。

- [ ] **Step 5: 執行測試確認通過**

Run: `cd app; npx vitest run src/services/__tests__/googleMaps.spec.js src/components/__tests__/mapPicker.spec.js`
Expected: PASS（兩檔皆綠，證明抽取未改變 `MapPicker` 行為）

- [ ] **Step 6: 提交**

```bash
git add app/src/services/googleMaps.js app/src/services/__tests__/googleMaps.spec.js app/src/components/MapPicker.vue
git commit -m "refactor(maps): extract shared google maps loader"
```

---

### Task 2: EventMap 唯讀多圖釘元件

**Files:**
- Create: `app/src/components/EventMap.vue`
- Test: `app/src/components/__tests__/eventMap.spec.js`

**Interfaces:**
- Consumes: `loadGoogleMaps()`（Task 1）
- Produces:
  - Props：`events: Array`（元素需 `id`、`title`、`coordinates`）、`selectedId: String|null`
  - Emits：`select`（事件 `id`）
  - Data：`ready: Boolean`、`marks: Array<{ id: string, marker: object }>`
  - DOM：容器 `.event-map`（`role="region"`、`aria-label="事件分佈圖"`）；無地圖時 `.event-map__note`

- [ ] **Step 1: 寫失敗測試**

建立 `app/src/components/__tests__/eventMap.spec.js`：

```js
import { mount } from '@vue/test-utils'
import EventMap from '../EventMap.vue'

const settle = () => new Promise((resolve) => setTimeout(resolve, 0))

function installGoogleMaps() {
  const created = []
  class FakeMap {
    constructor(_el, options) { this.options = options }
    setCenter() {}
  }
  class FakeMarker {
    constructor(options) { this.options = options; this.listeners = {}; created.push(this) }
    addListener(name, cb) { this.listeners[name] = cb }
    setMap(map) { this.map = map }
  }
  window.google = { maps: { Map: FakeMap, Marker: FakeMarker } }
  return { created }
}

const EVENTS = [
  { id: 'e-001', title: 'A', coordinates: { lat: 25.0455, lng: 121.509 } },
  { id: 'e-002', title: 'B', coordinates: null },
  { id: 'e-003', title: 'C', coordinates: { lat: 25.0237, lng: 121.521 } },
]

describe('EventMap', () => {
  afterEach(() => { delete window.google })

  test('無金鑰時顯示提示、ready 為 false、不丟例外', async () => {
    const w = mount(EventMap, { propsData: { events: EVENTS } })
    await settle()
    expect(w.find('.event-map__note').exists()).toBe(true)
    expect(w.vm.ready).toBe(false)
  })

  test('只為有座標的事件建立圖釘', async () => {
    installGoogleMaps()
    const w = mount(EventMap, { propsData: { events: EVENTS } })
    await settle()
    expect(w.vm.ready).toBe(true)
    expect(w.vm.marks.map((m) => m.id)).toEqual(['e-001', 'e-003'])
  })

  test('點圖釘會 emit select 帶對應 id', async () => {
    const { created } = installGoogleMaps()
    const w = mount(EventMap, { propsData: { events: EVENTS } })
    await settle()
    created[1].listeners.click()
    expect(w.emitted('select')[0]).toEqual(['e-003'])
  })

  test('events 變更時重建圖釘，不殘留舊的', async () => {
    installGoogleMaps()
    const w = mount(EventMap, { propsData: { events: EVENTS } })
    await settle()
    await w.setProps({ events: [EVENTS[0]] })
    await settle()
    expect(w.vm.marks.map((m) => m.id)).toEqual(['e-001'])
  })
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/components/__tests__/eventMap.spec.js`
Expected: FAIL（找不到 `../EventMap.vue`）

- [ ] **Step 3: 建立 `app/src/components/EventMap.vue`**

- 模板：容器 `.event-map`（`role="region"`、`aria-label="事件分佈圖"`）內含 `ref="canvas"`；`ready` 為 false 時顯示 `.event-map__note`（`role="status"`），文字說明地圖無法顯示、可用清單查看。
- `mounted`：`this.mapsLib = await loadGoogleMaps()`；有值才 `buildMap()`，否則 `ready=false`。
- `buildMap()`：`new mapsLib.Map(canvas, { center: ..., zoom: ... })`，再呼叫 `renderMarks()`。
- `renderMarks()`：先清空（對每個舊 mark 呼叫 `marker.setMap(null)`），再對每個 `hasCoordinates(event)` 的事件 `new mapsLib.Marker({ position: event.coordinates, map, title: event.title })`，`marker.addListener('click', () => this.$emit('select', event.id))`，收進 `this.marks`。
- `watch.events` → `renderMarks()`（mapsLib 尚未載入則略過）。
- `watch.selectedId` → 找到對應 mark 則 `map.setCenter(marker.getPosition())`（若 `getPosition` 存在）。
- `hasCoordinates(e)`：`e && e.coordinates && typeof e.coordinates.lat === 'number' && typeof e.coordinates.lng === 'number'`（不合成座標）。
- 樣式：`.event-map` 高度 ≥ 200px、`border-radius: var(--radius-card)`、暖色邊框；沿用 token。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/components/__tests__/eventMap.spec.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add app/src/components/EventMap.vue app/src/components/__tests__/eventMap.spec.js
git commit -m "feat(event-map): read-only multi-marker map with no-key fallback"
```

---

### Task 3: EventListView 版面與連動

**Files:**
- Modify: `app/src/views/query/EventListView.vue`
- Test: `app/src/views/query/__tests__/eventList.spec.js`

**Interfaces:**
- Consumes: `EventMap`（Task 2，`events`、`selectedId`、`@select`）
- Produces:
  - computed `locatedEvents`（帶數字座標者，維持原順序）、`unlocatedEvents`（無座標者）
  - data `selectedId: String|null`

- [ ] **Step 1: 寫失敗測試**

在 `app/src/views/query/__tests__/eventList.spec.js` 追加（沿用檔內既有的 `makeStore`／`mountView`／`flush`／`RouterLinkStub`）：

```js
const LOCATED = [
  { id: 'e-001', title: '有座標A', timeText: '今天', placeText: '中華路', status: 'reported', coordinates: { lat: 25.0455, lng: 121.509 } },
  { id: 'e-003', title: '有座標B', timeText: '今天', placeText: '汀州路', status: 'reported', coordinates: { lat: 25.0237, lng: 121.521 } },
  { id: 'e-004', title: '無座標C', timeText: '前天', placeText: '青年公園', status: 'reported', coordinates: null },
]

test('有座標者加編號、無座標者列最後並標示', async () => {
  vi.spyOn(eventService, 'list').mockResolvedValue(LOCATED)
  const w = mountView(makeStore())
  await flush()
  const badges = w.findAll('[data-event-num]').wrappers.map((n) => n.text())
  expect(badges).toEqual(['1', '2'])
  expect(w.text()).toContain('位置未標示')
  // 無座標事件排在最後
  const cards = w.findAll('.events__item').wrappers.map((c) => c.text())
  expect(cards[cards.length - 1]).toContain('無座標C')
})

test('EventMap 只收到有座標的事件', async () => {
  vi.spyOn(eventService, 'list').mockResolvedValue(LOCATED)
  const w = mount(EventListView, { localVue, store: makeStore(), stubs: { RouterLink: RouterLinkStub, EventMap: true } })
  await flush()
  const map = w.findComponent({ name: 'EventMap' })
  expect(map.props('events').map((e) => e.id)).toEqual(['e-001', 'e-003'])
})

test('EventMap 選取事件時，對應卡片標記為選中', async () => {
  vi.spyOn(eventService, 'list').mockResolvedValue(LOCATED)
  const w = mount(EventListView, { localVue, store: makeStore(), stubs: { RouterLink: RouterLinkStub, EventMap: true } })
  await flush()
  w.findComponent({ name: 'EventMap' }).vm.$emit('select', 'e-003')
  await flush()
  expect(w.find('.events__card--selected').text()).toContain('有座標B')
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/views/query/__tests__/eventList.spec.js`
Expected: FAIL（無 `[data-event-num]`、`.events__card--selected`、`位置未標示`）

- [ ] **Step 3: 修改 `EventListView.vue`**

- 匯入 `EventMap`，加入 `components`。
- `data` 加 `selectedId: null`。
- `computed` 加：
  - `locatedEvents()`：`this.events.filter(e => e.coordinates && typeof e.coordinates.lat === 'number' && typeof e.coordinates.lng === 'number')`
  - `unlocatedEvents()`：`this.events.filter(e => !this.locatedEvents.includes(e))`
- 模板：在清單前插入 `<EventMap :events="locatedEvents" :selected-id="selectedId" @select="selectedId = $event" />`。
- 清單改為先渲染 `locatedEvents`（每張卡片加 `<span class="events__num" data-event-num>{{ index + 1 }}</span>`，並在 `selectedId === event.id` 時加 `.events__card--selected`），再渲染 `unlocatedEvents`（加 `<span class="events__nolocation">位置未標示</span>`，無編號）。
- 既有篩選、錯誤、空狀態、`?q=` 行為不得改變。
- 樣式：手機 `.events__layout` 單欄（地圖在上、清單在下）；`@media (min-width: 1024px)` 改為兩欄（地圖左 `sticky`、清單右）。編號徽章用暖色 pill；`.events__card--selected` 用 `--accent`/`--accent-foreground`。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/views/query/__tests__/eventList.spec.js`
Expected: PASS（含既有測試）

- [ ] **Step 5: 提交**

```bash
git add app/src/views/query/EventListView.vue app/src/views/query/__tests__/eventList.spec.js
git commit -m "feat(events): add map+list layout with pin/card selection"
```

---

### Task 4: 全量驗證

**Files:** 無新增；驗證既有變更。

- [ ] **Step 1: 執行全部測試**

Run: `cd app; npm test`
Expected: 全部 PASS（含既有 `mapPicker.spec.js`、`eventList.spec.js` 與新測試）。

- [ ] **Step 2: 建置**

Run: `cd app; npm run build`
Expected: 成功產出 `dist/`，無錯誤。

- [ ] **Step 3: 人工複驗（需金鑰）**

在 `app/.env.local` 設 `VITE_GOOGLE_MAPS_KEY=<你的金鑰>`，於 375px / 1024px / 1440px 檢查：有金鑰時圖釘顯示於正確座標、點卡片高亮圖釘、點圖釘捲到卡片；無金鑰（暫時移除）時清單照常、地圖顯示提示。

- [ ] **Step 4: 提交（若有微調）**

```bash
git add -A
git commit -m "chore: events map+list verification fixes"
```

---

## Self-Review

- **Spec coverage**：§4.1 版面 → Task 3；§4.2 連動 → Task 2/3；§4.3 元件介面 → Task 1/2；§4.4 降級與邊界 → Task 2（無金鑰、無座標）+ Task 3（無座標列最後）；§4.5 無障礙 → Global Constraints + Task 2/3；§5 技術決策 → Task 1/2。
- **Step scan**：每步單一可檢查動作；程式碼塊只給測試與簽名，實作主體留給實作者。
- **Type consistency**：`loadGoogleMaps()`、`EventMap` props（`events`/`selectedId`）、emit（`select`）、data（`ready`/`marks`）、`locatedEvents`/`unlocatedEvents` 在 Task 1/2/3 一致。
- **Review Focus**：五條皆指派到 Task（1→Task 2、2→Task 2、3→Task 2、4→Task 2、5→Task 3）。
- **Proportion**：計畫長度與規格相當，未逐行轉錄程式。
