# 入口體驗視覺基準 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在守住高齡友善底線的前提下，把入口體驗（AppShell、導覽、首頁）精緻化為「任務優先」版面，並建立可延伸的字級／間距基準。

**Architecture:** 純前端視覺重構，不動資料層與路由邏輯。先在 `tokens.css` 建立字級／間距／字型 token 作為單一來源，接著把 `NAV_ITEMS`（桌面）與 `MOBILE_NAV_ITEMS`（手機，含首頁）改為帶 label 與 icon 的資料，導覽與首頁元件只消費這些資料與 token。`BigTaskCard` 以 `variant` prop 提供 hero／default 兩種層次，`HomeView` 以 hero 為主行動、兩張 default 為次要。

**Tech Stack:** Vue 2.7（Options API）、Vue Router 3、Vuex 3、Element UI 2、Vite 5、Vitest 1 + @vue/test-utils 1、SCSS/CSS 變數。

**Spec:** `docs/superpowers/specs/2026-10-05-entry-experience-baseline-design.md`

## Global Constraints

- 本次只做**淺色**；不得刪改既有 `.dark` token 區塊。
- 色彩一律使用 `tokens.css` 既有變數；**品牌橘 `#FF7139` 僅裝飾**，不可承載文字或當按鈕底（實測 2.74:1）。
- 全頁**單一強調色**（`--primary` 暖橘）；不得引入第二種強調色。
- 所有字級必須來自字級 token，不得 ad-hoc 寫死 px。
- 可點元素 ≥ 48×48px；卡片類 ≥ 88px（依 `app/ACCEPTANCE.md`）。
- 文字對比 ≥ WCAG AA（一般文字 4.5:1）；CTA 白字在 `#A44900` 上為 5.95:1。
- 導覽標籤（桌面）單行、導覽高度 ≤ 72px。
- 問候語一律中性；**禁止**假的使用者姓名／假數據。
- 尊重 `prefers-reduced-motion`。
- 測試指令：於 `app/` 目錄執行 `npm test`（=`vitest run`）。

## Review Focus

以下五類是規格隱含、但單元測試不易涵蓋、最可能在使用者手上出錯的情況。每條都在對應任務加上一個測試或靜態守門：

1. **鍵盤焦點被固定底部導覽遮住**（WCAG 2.4.11）——使用者 Tab 到畫面下方的元素時，焦點環不可被浮動導覽蓋掉。（Task 4）
2. **長中文標籤溢出**——在 375px 寬，較長的導覽標籤不可換行溢出。以限制標籤長度守住。（Task 2）
3. **極窄 320px 底部四項仍可點**——新增第 4 項後，每項仍維持 ≥48px 可點高度。（Task 4）
4. **reduced-motion 下卡片 hover 位移仍發生**——開啟減少動態時，卡片不得再位移。（Task 5）
5. **深色系統偏好下白字白底**——本次只做淺色，但不得讓未處理的深色偏好產生低對比文字。（Task 1）

---

### Task 1: 字級、間距與字型 token

**Files:**
- Modify: `app/src/styles/tokens.css`
- Test: `app/src/styles/__tests__/tokens.spec.js`

**Interfaces:**
- Produces（CSS 自訂屬性，供後續所有元件使用）：
  - 字級：`--font-size-display: 2.125rem`、`--font-size-h2: 1.5rem`、`--font-size-h3: 1.25rem`、`--font-size-body: 1.125rem`、`--font-size-meta: 1rem`
  - 行高：`--line-height-display: 1.2`、`--line-height-h2: 1.3`、`--line-height-h3: 1.35`、`--line-height-body: 1.75`、`--line-height-meta: 1.6`
  - 間距：`--space-1: 4px`、`--space-2: 8px`、`--space-3: 16px`、`--space-4: 24px`、`--space-5: 32px`、`--space-6: 48px`
  - 字型：`--font-sans`、`--font-latin`

- [ ] **Step 1: 寫失敗測試**

在 `app/src/styles/__tests__/tokens.spec.js` 末尾加入：

```js
test('包含字級、間距與字型 token', () => {
  expect(css).toContain('--font-size-display')
  expect(css).toContain('--font-size-meta')
  expect(css).toContain('--space-6')
  expect(css).toContain('--font-sans')
  expect(css).toContain('--font-latin')
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/styles/__tests__/tokens.spec.js`
Expected: FAIL（找不到 `--font-size-display`）

- [ ] **Step 3: 在 `tokens.css` 的 `:root` Base tier 之後新增區塊**

新增 `/* Type & spacing scale */` 區塊（值依 Interfaces 所列）。`--font-sans` 需含 CJK 系統字型 fallback（例如 `"Noto Sans TC", "PingFang TC", "Microsoft JhengHei", system-ui, sans-serif`）。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/styles/__tests__/tokens.spec.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add app/src/styles/tokens.css app/src/styles/__tests__/tokens.spec.js
git commit -m "feat(tokens): add type scale, spacing and font-family tokens"
```

---

### Task 2: 導覽資料（NAV_ITEMS / MOBILE_NAV_ITEMS）

**Files:**
- Modify: `app/src/router/index.js`
- Test: `app/src/components/__tests__/nav.spec.js`

**Interfaces:**
- Produces:
  - `export const NAV_ITEMS` — 桌面用，3 項，每項 `{ to: string, label: string, icon: string }`
  - `export const MOBILE_NAV_ITEMS` — 手機用，4 項（首頁在前），同結構
  - icon 名稱僅用：`'House' | 'Flag' | 'MapPin' | 'Gift'`

- [ ] **Step 1: 寫失敗測試**

在 `app/src/components/__tests__/nav.spec.js` 加入（import 路徑由測試檔望向 `src/router`）：

```js
import { NAV_ITEMS, MOBILE_NAV_ITEMS } from '../../router'

test('桌面 NAV_ITEMS 為 3 項且標籤已縮短', () => {
  expect(NAV_ITEMS.map((i) => i.label)).toEqual(['回報問題', '附近事件', '福利活動'])
  NAV_ITEMS.forEach((i) => expect(i.label.length).toBeLessThanOrEqual(4))
})

test('手機 MOBILE_NAV_ITEMS 含首頁共 4 項', () => {
  expect(MOBILE_NAV_ITEMS).toHaveLength(4)
  expect(MOBILE_NAV_ITEMS[0]).toEqual({ to: '/', label: '首頁', icon: 'House' })
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/components/__tests__/nav.spec.js`
Expected: FAIL（`MOBILE_NAV_ITEMS` 未定義／標籤不符）

- [ ] **Step 3: 改寫 `app/src/router/index.js` 的 NAV_ITEMS 區塊**

```js
export const NAV_ITEMS = [
  { to: '/report', label: '回報問題', icon: 'Flag' },
  { to: '/events', label: '附近事件', icon: 'MapPin' },
  { to: '/welfare', label: '福利活動', icon: 'Gift' },
]

export const MOBILE_NAV_ITEMS = [
  { to: '/', label: '首頁', icon: 'House' },
  ...NAV_ITEMS,
]
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/components/__tests__/nav.spec.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add app/src/router/index.js app/src/components/__tests__/nav.spec.js
git commit -m "feat(nav): shorten labels and add icon data for top/mobile nav"
```

---

### Task 3: TopNav 圖示與選中態

**Files:**
- Modify: `app/src/components/TopNav.vue`
- Test: `app/src/components/__tests__/nav.spec.js`

**Interfaces:**
- Consumes: `NAV_ITEMS`（Task 2）
- Produces: 每個 `[data-nav-item]` 內含一個 icon `<i>`；沿用 `.top-nav__link.router-link-active` 暖底樣式。

- [ ] **Step 1: 寫失敗測試**

在 `nav.spec.js` 加入：

```js
import { mount } from '@vue/test-utils'
import TopNav from '../TopNav.vue'

test('桌面導覽每項都有圖示', () => {
  const w = mount(TopNav, { stubs: ['router-link'] })
  const items = w.findAll('[data-nav-item]')
  expect(items).toHaveLength(3)
  expect(w.findAll('[data-nav-item] i')).toHaveLength(3)
  expect(w.text()).toContain('福利活動')
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/components/__tests__/nav.spec.js`
Expected: FAIL（`i` 數量為 0）

- [ ] **Step 3: 修改 `TopNav.vue`**

- 匯入 icon 元件（`House`/`Flag`/`MapPin`/`Gift`）並建立 `ICONS` 對照表。
- 在導覽最前面加入品牌（圓形 logo「社」+ 文字「社區資訊平台」），其餘導覽項靠右。
- `data()` 回傳 `NAV_ITEMS` 與 `ICONS`，或加 `methods: { iconFor(name) { return ICONS[name] } }`。
- 在 `<router-link>` 內，標籤前插入 `<component :is="iconFor(item.icon)" :size="20" aria-hidden="true" />`。
- 樣式：`.top-nav` `min-height` 不超過 72px；保持 `sticky`、單行。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/components/__tests__/nav.spec.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add app/src/components/TopNav.vue app/src/components/__tests__/nav.spec.js
git commit -m "feat(top-nav): add icons and tighten navigation height"
```

---

### Task 4: BottomNav 加入首頁與圖示

**Files:**
- Modify: `app/src/components/BottomNav.vue`
- Test: `app/src/components/__tests__/nav.spec.js`

**Interfaces:**
- Consumes: `MOBILE_NAV_ITEMS`（Task 2）
- Produces: 4 個 `[data-nav-item]`，第一個為首頁；每項 ≥48px。

- [ ] **Step 1: 寫失敗測試**

改寫 `nav.spec.js` 既有測試（原斷言 3 項與「找福利和活動」）為：

```js
test('手機底部導覽有四項且含首頁', () => {
  const w = shallowMount(BottomNav, { stubs: ['router-link'] })
  expect(w.findAll('[data-nav-item]')).toHaveLength(4)
  expect(w.text()).toContain('首頁')
  expect(w.text()).toContain('福利活動')
})

test('底部導覽每項都有圖示', () => {
  const w = mount(BottomNav, { stubs: ['router-link'] })
  expect(w.findAll('[data-nav-item] i')).toHaveLength(4)
})
```

同時把 `nav.spec.js` 頂部 `import BottomNav` 與 `shallowMount`/`mount` 匯入整理為一次匯入。

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/components/__tests__/nav.spec.js`
Expected: FAIL（數量為 3、無「首頁」）

- [ ] **Step 3: 修改 `BottomNav.vue`**

- 改匯入 `MOBILE_NAV_ITEMS`（取代 `NAV_ITEMS`）。
- 以同 Task 3 的方式加入 icon（`ICONS` 對照 + `<component :is>`，`size` 20）。
- 維持 `.bottom-nav__link` `min-height: 48px`；確認四項在 320px 仍各自可點（`flex: 1` 已滿足，勿改成 fixed 寬）。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/components/__tests__/nav.spec.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add app/src/components/BottomNav.vue app/src/components/__tests__/nav.spec.js
git commit -m "feat(bottom-nav): add home item and icons"
```

---

### Task 5: BigTaskCard 的 hero / default 變體

**Files:**
- Modify: `app/src/components/BigTaskCard.vue`
- Create: `app/src/components/__tests__/bigTaskCard.spec.js`

**Interfaces:**
- Produces:
  - Props：`variant`（`'default' | 'hero'`，預設 `'default'`）、`cta`（`string`，預設 `''`）
  - 根元素帶 `data-variant="<variant>"`；hero 且有 `cta` 時渲染 `.big-task-card__cta`

- [ ] **Step 1: 寫失敗測試**

建立 `app/src/components/__tests__/bigTaskCard.spec.js`：

```js
import fs from 'node:fs'
import { mount } from '@vue/test-utils'
import BigTaskCard from '../BigTaskCard.vue'

// 動態組路徑（同 tokens.spec.js）：避免 Vite 改寫 new URL 的相對路徑。
const cardSrc = fs.readFileSync(new URL(['..', 'BigTaskCard.vue'].join('/'), import.meta.url), 'utf8')

test('hero 變體顯示 CTA 與 data-variant', () => {
  const w = mount(BigTaskCard, {
    propsData: { to: '/report', title: '回報社區問題', variant: 'hero', cta: '開始回報' },
    stubs: ['router-link'],
  })
  expect(w.find('[data-variant="hero"]').exists()).toBe(true)
  expect(w.find('.big-task-card__cta').text()).toBe('開始回報')
})

test('default 變體不顯示 CTA', () => {
  const w = mount(BigTaskCard, {
    propsData: { to: '/events', title: '附近事件' },
    stubs: ['router-link'],
  })
  expect(w.find('[data-variant="default"]').exists()).toBe(true)
  expect(w.find('.big-task-card__cta').exists()).toBe(false)
})

test('保留 reduced-motion 位移守則', () => {
  expect(cardSrc).toContain('prefers-reduced-motion')
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/components/__tests__/bigTaskCard.spec.js`
Expected: FAIL（無 `data-variant`／`__cta`）

- [ ] **Step 3: 修改 `BigTaskCard.vue`**

- 加 `variant`、`cta` props。
- 根 `<router-link>` 加 `:data-variant="variant"`。
- `variant === 'hero'` 時：圖示放大（例 `:size="28"`）、加較大 padding、`cta` 非空時在 description 後渲染 `<span class="big-task-card__cta">{{ cta }}</span>`（非 `<button>`，因外層已是連結）。
- 保留既有 `prefers-reduced-motion` 區塊。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/components/__tests__/bigTaskCard.spec.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add app/src/components/BigTaskCard.vue app/src/components/__tests__/bigTaskCard.spec.js
git commit -m "feat(big-task-card): add hero and default variants"
```

---

### Task 6: HomeView 任務優先版面

**Files:**
- Modify: `app/src/views/home/HomeView.vue`
- Create: `app/src/views/home/__tests__/homeView.spec.js`

**Interfaces:**
- Consumes: `BigTaskCard`（Task 5，`variant="hero"` 與 `variant="default"`）、`AiPromptBar`
- Produces: 中性問候 `h1`「今天需要幫忙嗎？」；1 個 `[data-variant="hero"]`、2 個 `[data-variant="default"]`。

- [ ] **Step 1: 寫失敗測試**

建立 `app/src/views/home/__tests__/homeView.spec.js`：

```js
import { createLocalVue, mount } from '@vue/test-utils'
import Vuex from 'vuex'
import HomeView from '../HomeView.vue'
import report from '../../../store/modules/report'

const localVue = createLocalVue()
localVue.use(Vuex)

function render() {
  const store = new Vuex.Store({ modules: { report } })
  return mount(HomeView, { localVue, store, stubs: ['router-link'] })
}

test('採用任務優先版面：中性問候 + 主行動 CTA', () => {
  const w = render()
  expect(w.text()).toContain('社區資訊平台')
  expect(w.text()).toContain('今天需要幫忙嗎？')
  expect(w.find('[data-variant="hero"]').exists()).toBe(true)
  expect(w.text()).toContain('開始回報')
  expect(w.findAll('[data-variant="default"]')).toHaveLength(2)
})

test('問候不含假的使用者名稱', () => {
  const w = render()
  expect(w.text()).not.toMatch(/先生|小姐|王先生|李太太/)
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/views/home/__tests__/homeView.spec.js`
Expected: FAIL（找不到 hero／中性問候）

- [ ] **Step 3: 改寫 `HomeView.vue` 模板**

- 保留品牌列：頁首渲染一個含 logo（圓形「社」）與文字「社區資訊平台」的 header（`smoke.spec` 依賴此字串，且手機版需要品牌辨識）；以 `@media (min-width: 1024px){ .home__brand { display: none } }` 隱藏，因為桌面版品牌已移到 `TopNav`。
- 舊的 `.home__title` 改為中性問候 `h1`「今天需要幫忙嗎？」。
- `<BigTaskCard variant="hero" cta="開始回報" icon="Flag" title="回報社區問題" desc="看到路燈壞掉、垃圾沒收，拍照或直接描述都可以" to="/report" />`
- 兩張 `<BigTaskCard variant="default" ...>`：`MapPin`「附近事件」→ `/events`；`Gift`「福利活動」→ `/welfare`。
- 保留 `AiPromptBar` 表單與其事件綁定不變。
- 樣式：容器 `display:flex; flex-direction:column; gap: var(--space-3)`；桌面（≥1024px）`max-width: 760px; margin-inline: auto`；兩張 default 卡以 `grid-template-columns: 1fr 1fr` 排列，<768px 收成單欄。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/views/home/__tests__/homeView.spec.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add app/src/views/home/HomeView.vue app/src/views/home/__tests__/homeView.spec.js
git commit -m "feat(home): task-first entry layout with primary report action"
```

---

### Task 7: 字型載入

**Files:**
- Modify: `app/index.html`
- Modify: `app/src/styles/tokens.css`（若 Task 1 尚未含 `--font-sans` fallback 則於此補）
- Create: `app/src/styles/__tests__/fonts.spec.js`

**Interfaces:**
- Consumes: `--font-sans`、`--font-latin`（Task 1）
- Produces: `index.html` 載入 Noto Sans TC 與 Atkinson Hyperlegible。

- [ ] **Step 1: 寫失敗測試**

建立 `app/src/styles/__tests__/fonts.spec.js`：

```js
import fs from 'node:fs'
const htmlUrl = new URL(['..', '..', '..', 'index.html'].join('/'), import.meta.url)
const html = fs.readFileSync(htmlUrl, 'utf8')

test('載入 Noto Sans TC 與 Atkinson Hyperlegible', () => {
  expect(html).toContain('Noto+Sans+TC')
  expect(html).toContain('Atkinson+Hyperlegible')
})
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd app; npx vitest run src/styles/__tests__/fonts.spec.js`
Expected: FAIL

- [ ] **Step 3: 修改 `index.html`**

在 `<head>` 加入 `preconnect` 與樣式表連結（`display=swap` 已含於 Google Fonts URL）；於 `tokens.css` 設定 `--font-sans` / `--font-latin` 並在 `body` 套用。註解標明：正式上線建議自架字型（避免第三方請求）。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd app; npx vitest run src/styles/__tests__/fonts.spec.js`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add app/index.html app/src/styles/tokens.css app/src/styles/__tests__/fonts.spec.js
git commit -m "feat(fonts): load Noto Sans TC and Atkinson Hyperlegible"
```

---

### Task 8: 標記 navy MASTER 為 deprecated

**Files:**
- Modify: `design-system/default/MASTER.md`

**Interfaces:** 無程式介面；純文件。

- [ ] **Step 1: 在 `MASTER.md` 最上方加入 deprecation 標頭**

```markdown
> ⚠️ **DEPRECATED**：本檔的 navy 藍配色（`#0F172A` / `#0369A1`）已不採用。
> 全站視覺以 `Session5_設計方向_Firefox風格.md` 與 `design-system/firefox-acorn/` 的暖色 token 為準。
> 入口體驗的版面基準見 `docs/superpowers/specs/2026-10-05-entry-experience-baseline-design.md`。
```

- [ ] **Step 2: 驗證**

Run: `cd 'C:\Workspace\Project\web_dev\LCS_project'; git diff --stat design-system/default/MASTER.md`
Expected: 顯示 1 file changed，且 `grep DEPRECATED design-system/default/MASTER.md` 有輸出。

- [ ] **Step 3: 提交**

```bash
git add design-system/default/MASTER.md
git commit -m "docs: deprecate navy master design system in favour of firefox warm tokens"
```

---

### Task 9: 全量驗證

**Files:** 無新增；驗證既有變更。

- [ ] **Step 1: 執行全部測試**

Run: `cd app; npm test`
Expected: 全部 PASS（含既有 `appShell.spec.js`、`a11yStyles.spec.js`、`tokens.spec.js` 與新測試）。

- [ ] **Step 2: 建置**

Run: `cd app; npm run build`
Expected: 成功產出 `dist/`，無錯誤。

- [ ] **Step 3: 人工複驗（依 `app/ACCEPTANCE.md`）**

以瀏覽器於 375px / 768px / 1024px / 1440px 檢查：導覽單行、首頁主行動清楚、焦點環可見、reduced-motion 生效、無橫向捲動、無白字白底。

- [ ] **Step 4: 提交（若有微調）**

```bash
git add -A
git commit -m "chore: entry baseline verification fixes"
```

---

## Self-Review

- **Spec coverage**：範圍（§2）→ Task 1-8；設計稽核（§3）第 1 點→Task 8，第 2 點→Task 1，第 3 點→Task 5/6，第 4 點→Task 3/4，第 5 點→Task 7。無障礙（§5）→ Global Constraints + Review Focus。驗收（§8）→ 各 Task Step 與 Task 9。
- **Step scan**：每個 Step 為單一可檢查動作；程式碼區塊僅給簽名、測試與關鍵值，實作主體留給實作者。
- **Type consistency**：`variant`（`'default'|'hero'`）、`cta`、`NAV_ITEMS`/`MOBILE_NAV_ITEMS`、icon 名稱在 Task 2/3/4/5/6 一致。
- **Review Focus**：五條皆已指派到 Task（1→4、2→2、3→4、4→5、5→1）。
- **Proportion**：計畫長度與規格相當，未逐行轉錄程式。
