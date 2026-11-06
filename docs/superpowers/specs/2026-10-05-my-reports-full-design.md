# 「我的回報」完整版 — 設計規格

- 日期：2026-10-05
- 狀態：設計已核可，待規格複核
- 分支基準：`master`
- 目標分支：`feat/my-reports-full`
- 備註：本規格為第二份；第一份已提供「我的回報」最小版（`myReports` 記憶體 store + `/my-reports`）。

---

## 1. 背景與目的

第一份的「我的回報」只有最小可用：記憶體 store、基本清單、空狀態。

**目的**：把「我的回報」做**完整**——可**篩選狀態**、可**進詳情**、可**刪除單筆**、顯示**筆數**與更清楚的**空狀態**。維持**不持久化**（重整後清空）。

---

## 2. 範圍

### 2.1 納入（In scope）
- `myReports` store：快照擴充（加 `description`、`photo`）、新增 `remove(id)`。
- `/my-reports`：筆數、狀態篩選、每筆「查看／刪除」、兩種空狀態。
- 新增 `/my-reports/:id` 詳情頁（由快照顯示）。
- 刪除採**二次確認**（高齡友善，避免誤觸）。

### 2.2 排除（Out of scope）
- **持久化**（localStorage 等）：維持記憶體，重整後清空。
- 修改事件列表、三個助手對話、回報流程本身。
- 編輯已送出的回報、附件下載。

---

## 3. 現況

- `store/modules/myReports.js`：`state.list`（快照：`id/title/type/placeText/timeText/status/reportedAt`）、mutation `myReports/add`（同 id 只留最新、最新在前）。
- `views/my/MyReportsView.vue`：標題、空狀態（`EmptyState`）、以 `ResultCard` 列出、顯示類型。
- `ConfirmStep.confirm()`：送出成功後 `commit('myReports/add', event)`（event 來自 `eventService.report`，含 `description`、`photo`）。
- `router`：`/my-reports`（name `my-reports`）。

---

## 4. 目標設計

### 4.1 Store（`myReports`，維持記憶體）
- 快照擴充為：`{ id, title, type, placeText, timeText, status, reportedAt, description, photo }`（缺漏以空字串／`null` 補）。
- 新增 mutation：`remove(state, id)` — 由 `list` 移除該 id。
- `add` 行為不變（同 id 取代、最新在前）。

### 4.2 清單頁（`/my-reports`）
- **筆數**：標題下方顯示「共 N 筆」，N 為目前篩選後的筆數；當有套用篩選且 N 少於總數時，顯示「共 N 筆（全部 M 筆）」。
- **狀態篩選**：chips（全部／待處理／處理中／已完成）；預設「全部」。用語與事件列表一致（`reported→待處理`、`in-progress→處理中`、`resolved→已完成`）。篩選狀態為**頁面本地狀態**，不放 store。
- **每筆卡片**：標題、類型、地點、時間、狀態；兩個動作：
  - **查看** → `/my-reports/:id`。
  - **刪除** → 二次確認：第一次點變「確定刪除？」，再點才 `dispatch('myReports/remove', id)`；期間可取消。
- **空狀態**：
  - 完全沒有紀錄 → `EmptyState`「還沒有回報紀錄」＋前往回報（`/assistant/report`）。
  - 有紀錄但篩選後為空 → 顯示「這個狀態還沒有回報」，並提供切回「全部」。

### 4.3 詳情頁（`/my-reports/:id`）
- 由 `myReports.list` 以 `id` 找快照，顯示：標題、類型、狀態、地點、時間、描述、照片（若有；無則「這則回報沒有附照片」）。
- 找不到（已刪除／未知 id）→ 「這筆回報已移除」＋「回我的回報」連結（`/my-reports`）。
- 標題用 `h1`；「回我的回報」連結維持 ≥48px 與可見焦點。

### 4.4 無障礙
- 篩選 chips ≥48px、`aria-pressed` 標示選中；刪除為 `button`，二次確認有文字提示。
- 沿用入口基準 token；對比 AA、可見焦點、`prefers-reduced-motion`。
- 頁面同時有「回首頁」（AppShell）與「回我的回報」（詳情頁）。

---

## 5. 技術決策

1. **不持久化**：維持 `myReports` 記憶體 store；重整即清空（依使用者選擇）。
2. **詳情由快照顯示**：不依賴事件來源（重整／mock 重置後事件可能不存在），快照自足。
3. **篩選與刪除確認皆為頁面本地狀態**：不放 store。
4. **重用既有元件**：`ResultCard`、`EmptyState`、既有 chip 樣式。

---

## 6. 影響檔案

- Modify：`app/src/store/modules/myReports.js`
- Modify：`app/src/store/__tests__/myReports.spec.js`
- Modify：`app/src/views/my/MyReportsView.vue`
- Modify：`app/src/views/my/__tests__/myReportsView.spec.js`
- Create：`app/src/views/my/MyReportDetailView.vue`
- Create：`app/src/views/my/__tests__/myReportDetailView.spec.js`
- Modify：`app/src/router/index.js`（新增 `/my-reports/:id`）

---

## 7. 驗收標準

- [ ] `/my-reports` 顯示筆數。
- [ ] 狀態篩選可切換（全部／待處理／處理中／已完成），清單隨之更新。
- [ ] 「查看」進詳情，顯示描述與照片（若有）。
- [ ] 「刪除」需二次確認才真的移除；移除後清單與筆數更新。
- [ ] 完全沒有紀錄 → 「還沒有回報紀錄」＋前往回報；篩選後為空 → 「這個狀態還沒有回報」。
- [ ] 詳情找不到 id → 「這筆回報已移除」＋回清單連結。
- [ ] 整頁不出現 Agent／AI／API 等技術詞（G5）。
- [ ] 既有測試維持通過；`npm run build` 成功。

---

## 8. 後續（非本次）

- 跨重整的本機持久化（localStorage）與狀態更新。
- 編輯已送出的回報、附件管理、分享。

---

## 9. 參考

- 第一份規格：`docs/superpowers/specs/2026-10-05-assistant-flow-design.md`（§4.8 最小版）
- 現況：`app/src/store/modules/myReports.js`、`app/src/views/my/MyReportsView.vue`、`app/src/views/report/ConfirmStep.vue`
