# Session 5：設計方向 — Firefox（Acorn / Nova）風格

> 決策：採用 **Firefox 的設計語言（Acorn / Nova）**，不採用 Firefox 的「瀏覽器外框」；強調色由 navy 藍改為 **Firefox 暖色**。
>
> 產出：`design-system/firefox-acorn/tokens.css`（可直接匯入的 CSS 變數＋Tailwind v4 對應）。
>
> 適用範圍：全站。與 `Session3_介面規劃.md`、`Session4_UX規劃.md`、`design-system/default/MASTER.md` 併用；顏色與外觀以本檔為準。

---

## 1. Firefox 現況（研究結果，非印象）

| 名詞 | 是什麼 | 時間 |
|---|---|---|
| **Photon** | Firefox 57–88 的舊設計系統 | 2017 |
| **Proton** | Firefox 89 起的設計語言 | 2021 |
| **Acorn** | **目前官方設計系統**（Proton 之後更名）；定義 tokens／components／patterns | 現行 |
| **Nova** | Firefox 157 的最新改版（圓角泡泡分頁、浮動工具列、內容框選、暖色調、Compact Mode 回歸、主題選擇器） | 2026/09/29 |

**Acorn 的 token 架構（我們直接沿用）**
- 三層：**Base → Application → Component**
- 色彩用 `oklch()`，明暗用 CSS `light-dark()`
- 高對比用 `@media (prefers-contrast)` 與 `@media (forced-colors)`
- 圓角用 `--border-radius-*`（circle = 9999px）

**Firefox 品牌色（實際值）**
- Firefox Orange `#FF7139`、Accent Blue `#0062FA`、Ink `#15141A`
- Photon 規範明訂：**Firefox Orange 僅限品牌使用**；Orange 50 `#FF9400` 對比不足、Orange 60 `#D76E00` 僅達 AA Large

來源：`acorn.firefox.com`、Mozilla blog（Nova, 2026/09/29）、Firefox Photon color spec、designpieces/brandcolorcode。

---

## 2. 借用與避免（核心判斷）

**採用 Acorn/Nova 的「視覺語言」，不採用它的「密度」。**

| ✅ 借用 | ❌ 避免 |
|---|---|
| 圓角（卡片 16px、按鈕 pill） | Compact Mode、密集分頁 |
| 浮動分層：頁首/底部導覽與內容分離、柔陰影 | 多工具列、icon-only 小控制 |
| 內容框選：四周留白、不貼邊 | 邊到邊密集資訊 |
| 單一暖色強調 + 中性底 | 半透明/模糊造成的低對比「玻璃感」 |
| 淺/深主題、`prefers-contrast` 支援 | 用品牌橘 `#FF7139` 當文字或按鈕底 |
| 選中狀態的細微漸層 | 硬邊框、無分層的平面卡片 |
| 一致圓潤圖示（延續 Phosphor outline） | 混用填充/線性、混用字重 |

---

## 3. 色票（已用 WCAG 公式實測）

> 數值為實算對比；`PASS-AA` = ≥4.5:1（一般文字）。

### 3.1 淺色

| 角色 | Hex | 對比 | 用途 |
|---|---|---|---|
| 背景 Background | `#FAF7F4` | — | 暖紙色底 |
| 文字 Ink | `#15141A` | 18.31:1 | 標題、內文（Firefox Ink） |
| 卡片 Card | `#FFFFFF` | — | 浮動卡片 |
| 次要文字 Muted | `#5B5560` | 7.21:1 | 說明、meta |
| **主要 CTA Primary** | `#A44900` | 白字 **5.95:1** | 按鈕、連結、focus ring（Orange 70） |
| CTA Hover | `#863B00` | 白字 7.96:1 | hover / active |
| 選中底色 Accent | `#FFE8D1` | — | 選中 chip／nav 底 |
| 邊框 Border | `#E7E2DC` | — | 分隔線 |
| 成功 Success | `#1B7F3B` | 5.07:1 | 送出成功 |
| 錯誤 Destructive | `#C0211B` | 6.04:1 | 錯誤訊息 |
| 品牌橘 Brand | `#FF7139` | **2.74:1 ✗** | **僅裝飾／Logo，不可當文字或按鈕底** |

### 3.2 深色

| 角色 | Hex | 對比 |
|---|---|---|
| 背景 | `#1C1B22` | — |
| 文字 | `#FBFBFE` | 16.54:1 |
| 卡片 | `#2B2A33` | — |
| 次要文字 | `#B7B2BE` | 6.83:1（卡片上） |
| **Primary（暖）** | `#FF9F4D` | 8.39:1；**深字 `#1C1B22`** 在暖底上 8.39:1 |

### 3.3 高對比

依 Acorn 做法，於 `@media (prefers-contrast: more)` 把次要文字與邊框加深至純文字色；`@media (forced-colors: active)` 改用系統 `Highlight` / `CanvasText`。

> ⚠️ 品牌橘 `#FF7139` 實測 2.74:1，**不可**用於承載文字或當按鈕填色（白字會糊）。需要暖色 CTA 時一律用 `#A44900`（淺）／`#FF9F4D`（深）。

---

## 4. 形狀、間距、字型

| 項目 | 值 | 來源 |
|---|---|---|
| 控制項圓角 | `--radius: 12px` | Nova 圓潤化 |
| 卡片圓角 | `--radius-card: 16px` | 同上 |
| 按鈕／chip／nav | `--radius-pill: 9999px` | Acorn `border-radius-circle` |
| 浮動陰影 | `--shadow-float`（兩層柔陰影） | Nova 浮動分層 |
| 間距節奏 | `4 / 8 / 16 / 24 / 32 / 48`（寬鬆） | 高齡友善 |
| 基準字級 | `18px`（非 16px），行高 `1.75` | 高齡友善 |
| 中文字型 | `Noto Sans TC` | 繁中、字形完整 |
| 拉丁／數字 | `Atkinson Hyperlegible` | 高辨識度 |

---

## 5. 對應到元件（shadcn/ui + Tailwind）

| 元件 | Firefox-inspired 寫法 | 重點 |
|---|---|---|
| **Card** | `rounded-2xl border-0 shadow-float p-6` | 用陰影分層，不用硬邊框 |
| **Button（主要）** | `rounded-full h-12 px-6 text-lg` + `bg-primary text-primary-foreground` | pill、大、暖色 |
| **Button（次要）** | `rounded-full bg-secondary text-secondary-foreground` | 中性底 |
| **底部導覽** | 浮動 bar：`fixed bottom-4 inset-x-4 rounded-2xl shadow-float` + `min-h-14` | 與內容分離、不貼邊 |
| **頁首** | 浮動／內容 `mx-4 mt-4 rounded-2xl` 或透明，內容不貼邊 | 框選感 |
| **Input / Textarea** | `rounded-xl border bg-card min-h-12 text-lg` | 大、可讀 |
| **Filter Chip** | `rounded-full min-h-11 px-4`；選中 `bg-accent text-accent-foreground` | pill、48px |
| **Result Card** | Card + 左側色條或小圖示 | 來源/日期可掃讀 |
| **選中 nav** | `bg-accent text-accent-foreground` +（可選）極細暖色漸層 | Nova active 漸層 |
| **Focus ring** | `outline-2 outline-offset-2 outline-ring` | 暖色可見 |

---

## 6. 對無障礙的影響（必須守住）

方向可以時髦，但底線不能退：

- 暖色 CTA 一律用**已驗證**的 `#A44900`（淺）／`#FF9F4D`（深），不可用 `#FF7139`。
- 品牌橘只當裝飾；不承載資訊、不當文字。
- 浮動底部導覽是固定元素 → 需 `scroll-padding-bottom`，避免遮住鍵盤焦點（WCAG 2.4.11）。
- 圓角與陰影**不得**犧牲對比；邊框淡化時，卡片層次改由陰影與底色差表現，仍需 ≥3:1 的非文字對比。
- 深色模式須獨立驗證（不可直接沿用淺色值）。
- 尊重 `prefers-reduced-motion`；Nova 漸層僅為靜態裝飾，不做自動動畫。

---

## 7. 實作

設計 token 已備好：`design-system/firefox-acorn/tokens.css`

1. 匯入字型 → `@import` Tailwind → `@import` tokens → 用 `@theme inline` 映射（檔案底部有完整範例）。
2. 淺色為預設；深色用 `.dark` class（shadcn 相容）；`prefers-contrast` / `forced-colors` 已含。
3. 若要更貼近 Acorn，可把 `.dark` 換成 `light-dark()` + `color-scheme`（Acorn 原生做法）；`.dark` class 版本更能直接接 shadcn 的 `next-themes`。

---

## 8. 待決事項

1. `design-system/default/MASTER.md` 目前仍是 **navy 藍**。是否要**改寫為本 Firefox 暖色方向**（或改成 Firefox 版 MASTER）？
2. 深色模式是否列入 v1？（Acorn/Nova 兩者都支援，建議列為 v1.1）
3. 品牌橘 `#FF7139` 是否要用於 Logo／裝飾識別（需要 Logo 設計決策）？
4. 暖色 CTA `#A44900` 偏「赭紅／焦糖」，若想要更亮的橘感，需以**深字**（非白字）或加深色才能保對比——是否接受此取捨？

---

## 備註：查證方式

- 色票對比：以 WCAG 2.x 相對亮度公式實算（腳本 `contrast.py`），非憑感覺。
- Firefox 事實：Acorn 官方文件、Mozilla Nova 部落格、Photon 色彩規範。
- 若 Firefox 官方色票日後更新，請以 `acorn.firefox.com` 為準並重算對比。
