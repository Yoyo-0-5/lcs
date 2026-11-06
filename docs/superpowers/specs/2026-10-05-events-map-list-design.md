# 附近事件：地圖＋清單 — 設計規格

- 日期：2026-10-05
- 狀態：設計已核可，待規格複核
- 分支基準：`feat/entry-baseline`（本工作建立在其上，沿用入口基準的 token）
- 目標分支：`feat/events-map-list`

---

## 1. 背景與目的

「附近事件」頁（`/events`）目前是「篩選 chip ＋ 卡片清單」，資訊完整但**缺少位置感**，與「附近」這個名稱有落差。

**目的**：加入**唯讀、多圖釘的地圖**，與清單雙向連動，讓使用者一眼掌握事件分布，同時不犧牲高齡可及性（清單仍是主要介面）。

---

## 2. 範圍

### 2.1 納入（In scope）
- **新增 `EventMap.vue`**：唯讀、多圖釘的 Google Maps 元件。
- **抽出共用載入模組**：把 `MapPicker.vue` 內部的 `loadGoogleMaps` 抽成共用模組，`MapPicker` 改為使用它（避免兩處重複）。
- **改寫 `EventListView.vue`**：版面改為「地圖＋清單」，加入選取狀態與編號對應。
- 對應測試。

### 2.2 排除（Out of scope，後續另案）
- 圖釘叢集（clustering）、路線導航、依裝置位置自動置中。
- 地圖圖磚主題自訂、街景。
- 其他頁面。

---

## 3. 現況

- `MapPicker.vue`：Google Maps 封裝，**單一圖釘、可拖曳選位置**；金鑰來自 `import.meta.env.VITE_GOOGLE_MAPS_KEY`；**無金鑰時 no-op**，只顯示文字提示。
- 事件資料（`mocks/events.json`）：6 筆中 **5 筆帶 `coordinates`**，`e-004`（公園垃圾桶滿溢）為 `null`。
- `EventListView.vue`：`h1` ＋導覽語＋四組篩選 chip（時間／地區／類型／狀態）＋ `ResultCard` 清單（桌機兩欄）。
- 座標紀律：任何路徑都**不得合成座標**（`locationService`／`location` store 已明訂）。

---

## 4. 目標設計

### 4.1 版面
- **手機**：地圖在上（約 150px），清單在下。
- **桌面（≥1024px）**：地圖在左（約 45%、`sticky`），清單在右。
- 篩選 chip 維持在頁首。

### 4.2 圖釘與清單連動
- 兩者共用**編號**（1、2、3…）與狀態文字。
- 點清單卡片 → 對應圖釘高亮並置中。
- 點圖釘 → 捲動至對應卡片並高亮。

### 4.3 元件介面
- `EventMap.vue`
  - Props：`events`（陣列，需 `id`、`title`、`status`、`coordinates`）、`selectedId`（`String|null`）。
  - Emits：`select`（事件 `id`）。
  - 只渲染**有座標**的事件為圖釘；無座標者略過。
- 共用載入模組（建議 `app/src/services/googleMaps.js`）：匯出 `loadGoogleMaps()`，行為同現行（無金鑰／測試環境回 `null`，不注入真實 API）。

### 4.4 降級與邊界
- **無金鑰或載入失敗**：地圖容器顯示提示文字（`role="status"`），**不放圖釘、不丟例外**；清單照常完整。
- **無座標事件**：列在清單最後並標「位置未標示」，不放圖釘。
- **狀態**：載入骨架（地圖與清單同形）、空結果沿用 `EmptyState`、錯誤沿用 `ErrorAlert`。

### 4.5 無障礙
- **清單是主要介面**，包含所有資訊與操作；讀屏與鍵盤使用者只靠清單即可完成一切。
- **v1 不依賴地圖操作**：不新增圖釘的個別鍵盤焦點，也不要求使用者操作地圖；地圖純為視覺輔助。
- 地圖容器提供可讀的 `aria-label`（例如「事件分布圖」）描述用途，但不作為唯一資訊來源；不將其 `aria-hidden`（Google Maps 自身帶有語意）。
- 維持既有底線：48px 觸控目標、WCAG AA 對比、可見焦點、`prefers-reduced-motion`。

### 4.6 視覺
- 沿用 `feat/entry-baseline` 的字級／間距 token 與暖色票；卡片沿用 `ResultCard` 的資訊層次。

---

## 5. 技術決策

1. **地圖來源：Google Maps**（使用者已有 `VITE_GOOGLE_MAPS_KEY`，放入 `app/.env.local`）。
2. **抽出共用載入模組**：`MapPicker` 與 `EventMap` 共用同一載入器；測試環境不得注入真實 Google API。
3. **不合成座標**：只採用事件資料中既有的 `coordinates`。

---

## 6. 影響檔案

- Create：`app/src/components/EventMap.vue`
- Create：`app/src/services/googleMaps.js`
- Modify：`app/src/components/MapPicker.vue`（改用共用載入器）
- Modify：`app/src/views/query/EventListView.vue`
- Create：對應測試（`EventMap`、`EventListView`、載入器）

---

## 7. 驗收標準

- [ ] 有金鑰時，帶座標事件正確顯示為多個圖釘。
- [ ] 無金鑰／載入失敗時，清單照常完整，地圖區顯示提示且不丟例外。
- [ ] 點卡片高亮對應圖釘；點圖釘捲動並高亮對應卡片。
- [ ] 無座標事件列於清單最後並標「位置未標示」，無圖釘。
- [ ] 讀屏使用者不需地圖即可取得全部事件資訊。
- [ ] 既有測試（`vitest`）維持通過；`npm run build` 成功。

---

## 8. 後續（非本次）

- 圖釘叢集、依裝置位置自動置中、地圖圖磚主題、事件詳情頁的地圖顯示。

---

## 9. 參考

- 視覺 mock：`.superpowers/brainstorm/events-layout/content/event-map-list-detail.html`
- 相關實作：`app/src/components/MapPicker.vue`、`app/src/views/query/EventListView.vue`、`app/src/mocks/events.json`
- 入口基準：`docs/superpowers/specs/2026-10-05-entry-experience-baseline-design.md`
