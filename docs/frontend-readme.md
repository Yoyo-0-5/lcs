# 社區資訊整合平台（LCS Project）

以 **Web 為主要操作介面**的社區資訊整合平台，讓里民可以**申報社區事件**、**查詢目前事件**、以及用自然語言**搜尋福利與活動**。以**高齡里民作為主要設計基準**，所有里民皆可使用。

> 目前為 **課程／展示用 MVP**：純前端 SPA ＋ 本地 mock 資料，**尚無後端／真 LLM／真 multi-agent**（見「範圍與已知限制」）。

---

## 功能

- **申報社區事件**：描述發生什麼事 → 確認位置（定位／文字／地圖，含「不確定先送出」的 §14.4 降級）→ 摘要確認 → 送出。支援**匿名**、**語音輸入**、**地圖**、**上傳照片／音檔**與手機**拍照**。
- **查詢目前事件**：預設顯示近期且相關的事件，提供時間／地區／類型／狀態**篩選 chips**、事件詳情、狀態標籤（待處理／處理中／已完成）。
- **搜尋福利與活動**：自然語言輸入 → 三段式載入回饋 → 結果卡（來源／發布日／活動日／原始連結）；找不到時明確告知，不產生幻覺內容。
- **首頁意圖分流**：萬用輸入框會自動判斷要進「申報／查詢／福利」流程；無法判斷時以 P12 二選一釐清。
- **響應式（mobile-first）**：手機＝底部浮動導覽＋單欄；桌機（1024+）＝頂部導覽＋適合處雙欄。
- **高齡友善與無障礙**：18px 基準字級、行高 1.75、觸控目標 ≥48px、可見焦點框、`prefers-reduced-motion`、對比 ≥4.5:1。

---

## 技術棧

| 層 | 選擇 |
|---|---|
| 框架 | Vue **2.7** |
| 建置 | Vite（`@vitejs/plugin-vue2`） |
| 套件管理 | pnpm（亦可用 npm / yarn；見「快速開始」） |
| 元件庫 | Element UI 2.15.x（SCSS 客製主題，圖示全用 Element UI icons） |
| 路由 | Vue Router 3 |
| 狀態 | Vuex 3 |
| 資料層 | axios ＋ axios-mock-adapter ＋ 本地 JSON（`USE_MOCK` 可替換接縫） |
| 地圖 | Google Maps JS API |
| 語音 | Web Speech API（`zh-TW`） |
| 測試 | Vitest（`@vue/test-utils`） |

---

## 快速開始

需求：**Node 18 以上**。指令以最普及的 **npm** 為主；若你慣用 pnpm / yarn，見下方替代。

```bash
cd app
npm install
npm run dev        # 開發伺服器，預設 http://localhost:5173
```

<details>
<summary>使用其他套件管理器</summary>

- **pnpm**：`cd app && pnpm install && pnpm dev`
  （或在專案根目錄：`pnpm -C app install`、`pnpm -C app dev`）
- **Corepack**（Node 內建，**不需先安裝 pnpm**）：先執行 `corepack enable pnpm`，之後即可直接用 `pnpm`。
- **yarn**：`cd app && yarn && yarn dev`
- **npm（從專案根目錄）**：`npm --prefix app install`、`npm --prefix app run dev`

</details>

---

## 安裝 pnpm（給想用 pnpm 的開發者）

本專案以 **pnpm 11** 開發。以下任選一種安裝方式，裝完用 `pnpm -v` 確認版本。

### 方法 1：Corepack（推薦；Node 內建，免額外安裝）

Node 16.9 以上（含 18／20／22／24）都內建 Corepack，可直接啟用 pnpm：

```bash
corepack enable                         # 啟用 pnpm / yarn 的 shim
corepack prepare pnpm@11 --activate     # 指定版本（或改用 @latest）
pnpm -v
```

> Windows 若因權限失敗，請改用「**以系統管理員身分開啟**的終端機」再執行 `corepack enable`，或改用方法 2。

### 方法 2：用 npm 全域安裝（最簡單）

```bash
npm install -g pnpm
pnpm -v
```

### 方法 3：官方獨立安裝腳本

- **Windows（PowerShell）**

  ```powershell
  iwr https://get.pnpm.io/install.ps1 -useb | iex
  ```

- **macOS / Linux**

  ```bash
  curl -fsSL https://get.pnpm.io/install.sh | sh -
  ```

  （macOS 也可用 `brew install pnpm`。）

### 不想安裝也能用（npx）

不裝、只用一次：

```bash
npx pnpm@11 -C app install
npx pnpm@11 -C app dev
```

裝好（或使用 npx）後，於 `app/` 目錄使用：

```bash
pnpm install
pnpm dev
```

> **不想安裝 pnpm？** 沒問題——本專案用 **npm 也能跑**，見上方「快速開始」。

---

## 指令

以下指令均**在 `app/` 目錄內**執行（或於根目錄用 `npm --prefix app <script>`／`pnpm -C app <script>`）。

| 動作 | npm | pnpm |
|---|---|---|
| 安裝依賴 | `npm install` | `pnpm install` |
| 開發伺服器 | `npm run dev` | `pnpm dev` |
| 建置（輸出 `app/dist/`） | `npm run build` | `pnpm build` |
| 預覽建置結果 | `npm run preview` | `pnpm preview` |
| 執行測試 | `npm test` | `pnpm test` |

---

## 環境變數

地圖（定位顯示、地圖點選位置）使用 Google Maps Platform，需要 API 金鑰。於 `app/.env.local` 設定：

```
VITE_GOOGLE_MAPS_KEY=你的金鑰
```

> 未設定金鑰時，地圖會**優雅降級**（顯示提示），不影響其他流程與測試。

---

## Demo 情境開關

在網址加上 `?mock=` 可強制觸發各種狀態，方便現場展示：

| 參數 | 效果 |
|---|---|
| `?mock=empty` | 搜尋／查詢無結果 |
| `?mock=timeout` | 搜尋逾時（顯示「繼續等待／重新搜尋」） |
| `?mock=partial` | 部分來源失敗（先給可靠結果＋略過提示） |
| `?mock=error` | 服務錯誤（顯示錯誤與重試） |
| `?mock=ambiguous` | 萬用輸入一律走 P12 釐清 |

例：`http://localhost:5173/welfare?mock=timeout`

---

## 專案結構

```text
.
├── app/                     # Vue 2.7 前端應用
│   ├── src/
│   │   ├── views/           # 頁面（home / report / query / welfare / clarify）
│   │   ├── components/      # 共用元件（AiPromptBar、ResultCard、附件、icons…）
│   │   ├── services/        # 服務層（intent / event / welfare / location / speech；USE_MOCK 接縫）
│   │   ├── store/           # Vuex 模組（report / query / welfare / location / intent）
│   │   ├── mocks/           # 本地 JSON 與 demo 情境
│   │   ├── api/             # axios instance 與 mock adapter
│   │   └── styles/          # tokens.css（Firefox Acorn/Nova 暖色）＋ Element 覆寫
│   ├── ACCEPTANCE.md        # S4 §12 無障礙交付檢查清單
│   └── package.json
├── design-system/           # 設計 tokens 與設計準則
├── Session1~8_*.md          # 各階段規劃與設計文件（見下）
└── README.md
```

---

## 測試

```bash
cd app
npm test
```

涵蓋服務層（intent／event／welfare／location）、store、共用元件、三支流程與無障礙。目前為 **37 檔 / 273 測試**。

---

## 設計與規劃文件

| 文件 | 內容 |
|---|---|
| `Session1_產品目標與範圍.md` | 產品定位、主要使用者、核心需求 |
| `Session2_使用情境與互動需求.md` | 使用情境、互動原則 |
| `Session3_介面規劃.md` | 畫面清單、wireframe、元件、tokens |
| `Session4_UX規劃.md` | 流程、狀態、回饋、文案、無障礙、測試 |
| `Session5_設計方向_Firefox風格.md` | Firefox Acorn/Nova 視覺語言與色票 |
| `Session6_使用者流程設計.md` | 三功能的端到端使用者流程 |
| `Session7_系統架構.md` | 系統架構、技術棧、服務接縫 |
| `Session8_實作計畫.md` | 實作計畫（任務分解） |

---

## 範圍與已知限制

- **純前端 ＋ 本地 mock**：沒有後端、真 LLM、真 multi-agent、真網頁檢索。所有「智慧」與外部服務都包在 `services/` 的 `USE_MOCK` 接縫後，未來可無痛接真後端。
- **Vue 2 已 EOL**（2023/12/31），本專案依指定技術棧沿用。
- **Google Maps 需要 API 金鑰**；未設時地圖降級。
- `app/ACCEPTANCE.md` §5 有 **4 項需真實瀏覽器**複驗：375px 像素重排、對比實測、Tab 焦點順序、`prefers-reduced-motion` 執行時行為。
- 若以**靜態主機**部署 `app/dist`，因使用 **history 路由**，需設定 SPA fallback（將未知路徑回傳 `index.html`）。

---

## 授權

尚未指定（課程／個人專案）。
