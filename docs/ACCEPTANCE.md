# ACCEPTANCE — 無障礙與 375px 驗收（Task 23）

- 分支：`feature/community-mvp`
- 範圍：S4 §12 UX 交付檢查清單、G4 無障礙底線、G5 文案、G6 響應式。
- 環境：Windows + pnpm（Vitest／jsdom）。無瀏覽器可用，需實際渲染的項目另列「待瀏覽器複驗」。
- 驗證指令：`pnpm -C app test`、`pnpm -C app build`。

**狀態圖例**：`[x]` 已驗證（靜態檢視＋單元測試）；`[ ]` 待瀏覽器人工複驗（附後續）。

---

## 1. S4 §12 交付檢查清單

- [x] **每個畫面的 Default／Loading／Empty／Error／Partial／Success 都已設計** — 見 §2 狀態矩陣；各狀態皆有對應元件（`SearchingState`／`EmptyState`／`ErrorAlert`／`NoResultState`）。
- [x] **所有空／錯狀態都有可執行的下一步** — `EmptyState`／`ErrorAlert` 皆以 `action` slot 提供下一步；本任務補上「位置文字搜尋失敗」的 `再試一次`（`LocationStep.vue`）。
- [x] **一次只問一個缺少欄位** — 申報流程依序只問「發生什麼事」→「位置」；時間／照片為選填，於摘要頁顯示不追問。
- [x] **位置流程具備文字描述替代路徑；地圖有非拖曳操作** — `LocationStep` 提供「用文字描述」且各模式皆可切回；`MapPicker` 支援點擊放置與方向鍵移動圖釘（`mapPicker.spec.js`）。
- [x] **固定底部導覽以 `scroll-padding-bottom` 補償焦點（WCAG 2.4.11）** — `app/src/styles/element-override.scss`；桌機另有 `scroll-padding-top`。由 `a11yStyles.spec.js` 守門。
- [x] **每筆外部結果都附來源／日期／原始連結；AI 內容有揭露** — `ResultCard`＋`SourceBadge`（來源／發布日）＋原始公告連結；`ResultList` 有「以下內容由系統整理，請以原始來源為準。」。`welfareView.spec.js` 逐筆斷言來源／日期／連結。
- [x] **搜尋等待有分段回饋，逾時有出路** — `SearchingState` 三段文案＋逾時「繼續等待／重新搜尋」；`welfareView.spec.js` 驗證。
- [x] **文案經 375px、放大字級檢視（靜態）** — 無固定寬度 >375px 的容器；chip／卡片皆 `flex-wrap`／單欄；`confirm__row` 以 `1fr`＋`word-break` 收斂。像素級渲染待 §4 複驗。
- [x] **對比、focus、觸控 ≥48px、reduced-motion 靜態具備** — 對比：`tokens.css` 逐 token 標註 WCAG 比值（內文 ≥4.5:1）；focus：`:focus-visible` 使用 `--ring`；觸控：互動元件 `min-height:48px` 起（見 §3 清單）；reduced-motion：`element-override.scss` 全域守則＋`BigTaskCard`／`ClarifyView` 局部守則。由 `a11yStyles.spec.js` 守門。**此列僅代表靜態樣式存在；對比渲染量測與 reduced-motion 執行時行為待 §5 複驗，尚未計為通過。**
- [x] **畫面不出現 Agent／API／Database 等技術詞（G5）** — 全 `views/`／`components/` 掃描無命中；技術詞僅存在於程式註解與測試。
- [x] **語音、定位被拒時，文字路徑完整可用** — 語音不支援／失敗時 `NlInputBox` 顯示「請直接用打字」；定位拒權時 `LocationStep` 自動改走文字描述（`locationStep.spec.js` 兩者皆有測試）。

## 2. 每畫面狀態矩陣

| 畫面（路由） | Default | Loading | Empty | Error | Partial | Success |
|---|---|---|---|---|---|---|
| 首頁 `/` | ✓ | ✓「正在理解…」 | n/a | n/a | n/a | ✓ 導向流程 |
| 描述 `/report` | ✓ | ✓ 送出中 | n/a | n/a（純解析） | n/a | → 位置 |
| 位置 `/report/location` | ✓ idle/text/map | ✓ 定位中 | n/a | ✓＋再試一次 | n/a | → 確認 |
| 確認 `/report/confirm` | ✓ | ✓ 送出中 | n/a | ✓＋再試一次 | n/a | → 完成 |
| 完成 `/report/done` | n/a | n/a | n/a | n/a | n/a | ✓ |
| 事件列表 `/events` | ✓ | ✓ | ✓＋回報 | ✓＋再試一次 | n/a | ✓ |
| 事件詳情 `/events/:id` | ✓ | ✓ | ✓ 已移除＋回列表 | ✓＋再試一次 | n/a | ✓ |
| 福利 `/welfare` | ✓ 輸入 | ✓ 三段式 | ✓ 無結果＋重搜/放寬 | ✓＋再試一次 | ✓ 標示略過 | ✓ |
| 釐清 `/clarify` | ✓ 兩層 | ✓ 選項停用 | n/a | n/a | n/a | → 流程 |

（`n/a` 表示該畫面無此種資料狀態；皆經原始碼檢視確認。）

## 3. 觸控目標 ≥48px（靜態清單）

- 導覽：`TopNav`／`BottomNav` 連結 `min-height:48px`。
- 首頁／申報／福利主按鈕 `min-height:56px`；`VoiceButton`／chip／`ResultCard` 連結 ≥48px。
- 卡片類（`BigTaskCard`／`ClarifyView` 選項）≥88px；`MapPicker` 畫布 280px；`NlInputBox` 多行框 120px。

## 4. 鍵盤走查（Step 2）

以原始碼確認三支流程的焦點可及性：

- 焦點陷阱：無 `el-dialog`／modal；無攔截 `Tab`／`keydown` 的全域處理器；`MapPicker` 僅攔截方向鍵（`preventDefault` 限方向鍵）。因此 **導覽不會困住焦點**。
- 走查順序（DOM 序）：首頁大卡片＋輸入＋開始 → 申報描述／位置／確認按鈕 → 查詢 chip／卡片連結 → 福利輸入／範例／結果連結。
- `Escape`：無需關閉的浮層，行為中性。
- [ ] **待瀏覽器複驗**：真實 Tab 焦點順序、`scroll-padding` 於捲動時是否避免底部導覽遮住焦點。後續：以 Chrome／Firefox 於 375px 手動 Tab 三支流程。

## 5. 待瀏覽器複驗（無法在無瀏覽器環境確認）

- [ ] 375px／200% 放大字級的**像素級**不破版（已做靜態檢視，需渲染截圖佐證）。
- [ ] 以量測工具（如 axe／Lighthouse）實測**對比**渲染值。
- [ ] `prefers-reduced-motion` 於系統設定下的實際動效（樣式與測試已具備）。

## 6. 本任務修正

- `NlInputBox.vue`：輸入框以 `aria-describedby` 關聯語音提示／錯誤訊息。
- `MapPicker.vue`：畫布以 `aria-describedby` 關聯操作說明（不支援提示／方向鍵提示）。
- `LocationStep.vue`：文字搜尋失敗的 `ErrorAlert` 補上「再試一次」下一步與樣式。
- `element-override.scss`：新增全域 `prefers-reduced-motion: reduce` 守則。
- 測試：`components/__tests__/a11y.spec.js`、`styles/__tests__/a11yStyles.spec.js`、`locationStep.spec.js`（+9 tests）。

## 7. 驗證結果

- `pnpm -C app test`：**36 files, 250 tests passed**（原 241，+9）。
- `pnpm -C app build`：**built in ~9s，成功**（僅既有 chunk size 警告）。
