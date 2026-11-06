# 入口體驗視覺基準 — 設計規格

- 日期：2026-10-05
- 狀態：設計已核可，待規格複核
- 範圍：入口體驗（AppShell、導覽、首頁）視覺精緻化，並建立可延伸的視覺基準

---

## 1. 背景與目的

**現況**：`LCS_project`（社區資訊平台）功能完整、可及性已做到位（三層 token、WCAG 對比實算、48px 觸控目標、`prefers-contrast`/`forced-colors`/`prefers-reduced-motion`），但視覺停在「能用」而非「被設計過」。

**目的**：在不犧牲高齡友善底線的前提下，把入口體驗精緻化，並建立一套後續頁面可依循的視覺基準。

**已定決策**：
1. 方向：**精緻化現有 Firefox / Acorn 暖色語言**（不更換視覺語言）。
2. 版面：**「任務優先」（Option B）** — 以「回報社區問題」為主行動的視覺重心。

---

## 2. 範圍

### 2.1 本次納入（In scope）
- `app/src/components/AppShell.vue` — 版面與間距基準
- `app/src/components/TopNav.vue` — 桌面導覽（加圖示、短標籤、選中態）
- `app/src/components/BottomNav.vue` — 手機浮動導覽（加圖示、新增「首頁」）
- `app/src/views/home/HomeView.vue` — 任務優先版面
- `app/src/components/BigTaskCard.vue` — 卡片視覺層次
- `app/src/components/AiPromptBar.vue` — 視覺對齊（結構不動）
- `app/src/styles/tokens.css` — 新增字級／間距 token
- `app/src/router/index.js` — `NAV_ITEMS` 標籤與 icon 對應

### 2.2 本次排除（Out of scope，後續另案）
- **深色模式**：`Session5` 原本即列為 v1.1；本次基準先做淺色，深色沿用同一組 token 後補。
- **其他頁面**（福利查詢、事件、通報流程、完成頁、clarify）：待本基準成立後照基準套用。
- **資料層、路由守衛、服務邏輯**：不動。

---

## 3. 現況問題（設計稽核）

1. **兩套設計系統衝突**：`design-system/default/MASTER.md` 是 navy 藍（`#0F172A`/`#0369A1`），與實作使用的暖色 token（`firefox-acorn`）不一致。`Session5` 第 8 節已列此為待決事項。
2. **無字級節奏**：字級散落各元件（1rem / 1.125 / 1.25 / 1.375 / 1.75），無統一 scale。
3. **卡片同質**：`BigTaskCard` 與 `ResultCard` 共用同一種陰影與邊框，缺乏主次。
4. **導覽陽春**：`TopNav`/`BottomNav` 僅純文字，已有的 icon 組未使用；手機導覽沒有回首頁的路徑。
5. **字型未明確載入**：文件載明 Atkinson Hyperlegible + Noto Sans TC，但元件皆以 `inherit` 繼承，未見明確載入點。

---

## 4. 目標設計

### 4.1 版面（任務優先）
```
[ 品牌列 ]
今天需要幫忙嗎？                          ← H1，中性問候（不可假個人化）
┌─────────────────────────────────┐
│ [icon]  回報社區問題              │      ← 主行動卡（視覺重心）
│         描述…　[ 開始回報 ]       │
└─────────────────────────────────┘
┌───────────────┐ ┌───────────────┐
│ [icon] 附近事件 │ │ [icon] 福利活動 │      ← 兩張次要卡
└───────────────┘ └───────────────┘
[ 或直接說你想做什麼…             ▶ ]      ← AI 輸入列
```
- 問候語一律中性（例如「今天需要幫忙嗎？」）。**禁止**假的使用者姓名／個人化。
- 桌面內容置中收斂（建議 `max-width` 約 760px）。

### 4.2 導覽
- **桌面 `TopNav`**：品牌（logo + 名稱）＋「圖示 + 標籤」pill；選中為暖底（`--accent` / `--accent-foreground`）；`sticky`；高度 ≤ 72px；單行。
- **手機 `BottomNav`**：浮動 pill bar；圖示 + 標籤；**新增「首頁」**；選中暖底；沿用 `--shadow-float-strong`。
- **標籤縮短**：`回報問題` / `附近事件` / `福利活動`（原為「看看附近事件」「找福利和活動」）。
- icon 對應：首頁 `House`、回報 `Flag`、事件 `MapPin`、福利 `Gift`（沿用現有 icon 元件）。

### 4.3 字級與間距 token（新增於 `tokens.css`）
| 角色 | 大小 / 行高 | 字重 |
|---|---|---|
| Display / H1 | 34px / 1.2 | 800 |
| H2 | 24px / 1.3 | 700 |
| H3 | 20px / 1.35 | 700 |
| Body | 18px / 1.75 | 400 |
| Meta | 16px / 1.6 | 400 |

- 間距節奏：`4 / 8 / 16 / 24 / 32 / 48`
- 圓角：控制項 `12px`、卡片 `16px`、按鈕／chip `9999px`

### 4.4 元件
- **主要 CTA**：pill、高度 ≥ 52px、`--primary`（`#A44900`）、白字（對比 5.95:1）。
- **次要按鈕**：中性底（`--secondary`）。
- **篩選 chip**：pill；選中 `--accent` / `--accent-foreground`。
- **焦點環**：`outline` 用 `--ring`。
- **AI 輸入列 (`AiPromptBar`)**：結構與行為不動，只做視覺對齊（沿用現有 `2px` 邊框與 focus-within）。
- **狀態**：載入（骨架與最終版型同形）、空狀態、錯誤（行內、可行動），沿用現有 `EmptyState`/`ErrorAlert`/`SearchingState` 並對齊視覺。

### 4.5 色彩
- 一律使用 `tokens.css` 既有值。
- 品牌橘 `#FF7139` **僅裝飾**（2.74:1，不可承載文字或按鈕底）。
- 全頁單一強調色，不得出現第二種強調色。

---

## 5. 無障礙底線（不可退讓）

- 所有可點元素 ≥ 48×48px。
- 文字對比 ≥ WCAG AA（一般文字 4.5:1）。
- 可見鍵盤焦點（`--ring`）。
- 尊重 `prefers-reduced-motion`。
- 固定的底部導覽需以 `scroll-padding-bottom` 避免遮住焦點。
- 標題層級正確（`h1` 單一）。

---

## 6. 已定決策

1. **`design-system/default/MASTER.md`（navy 版）**：標記為 deprecated，頂端加註指向 `Session5_設計方向_Firefox風格.md` 與 `design-system/firefox-acorn/`，避免兩套設計系統並存造成混淆。
2. **字型載入**：明確載入 Noto Sans TC（中文）＋ Atkinson Hyperlegible（拉丁／數字），設定系統字型 fallback 與 `font-display: swap`。
3. **深色模式**：本次不做，列為 v1.1；本次只保留既有 `.dark` token 不破壞。
4. **版型收斂寬度**：桌面首頁內容 `max-width: 760px` 置中。

---

## 7. 影響檔案

- `app/src/components/AppShell.vue`
- `app/src/components/TopNav.vue`
- `app/src/components/BottomNav.vue`
- `app/src/components/BigTaskCard.vue`
- `app/src/components/AiPromptBar.vue`
- `app/src/views/home/HomeView.vue`
- `app/src/router/index.js`
- `app/src/styles/tokens.css`

---

## 8. 驗收標準

- [ ] 桌面導覽為單行、高度 ≤ 72px；選中態使用暖底。
- [ ] 手機導覽含「首頁」，選中態使用暖底。
- [ ] 首頁有明確主行動（回報），視覺重心清楚。
- [ ] 所有字級皆來自新增的 type token，無 ad-hoc 字級。
- [ ] 所有 CTA 對比達 AA，且桌面標籤不換行。
- [ ] `prefers-reduced-motion` 下無裝飾性動畫。
- [ ] 既有測試（`vitest`）維持通過。
- [ ] 無假個人化文字、無假數據。

---

## 9. 參考

- 視覺輔助畫面（本次討論）：`.superpowers/brainstorm/entry-baseline/content/entry-directions.html`、`optionB-detail.html`
- 設計方向：`Session5_設計方向_Firefox風格.md`
- Token 實作：`app/src/styles/tokens.css`、`design-system/firefox-acorn/tokens.css`
