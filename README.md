# 社區小尖兵（Community Envoy）— 整合型 Monorepo

社區小尖兵包含重寫後的前端（`frontend/`）與 Python／FastAPI＋LangGraph AI 後端（`backend/`）。前端是原生 HTML／CSS／JavaScript＋Vite，透過 `/api` 與後端串接；舊的 Vue 前端（`frontend-old/`）已不再部署。

## 目錄結構

```text
.
├── frontend/            # 新前端（靜態 HTML/CSS/JS＋Vite）
│   ├── index.html
│   ├── pages/           # 聊天與社區事件申報頁面
│   ├── js/              # 申報流程、聊天、定位、天氣
│   ├── js/api/          # API 呼叫與執行期設定
│   ├── vite.config.js   # /api、/uploads 開發代理與靜態建置
│   └── Dockerfile       # Nginx 靜態服務＋API 反向代理
│
├── backend/             # FastAPI＋LangGraph AI Agent
│   ├── server.py        # 啟動檔（uvicorn，含 HTTPS 憑證自動偵測）
│   ├── app/server.py    # API 路由與前端 dist 靜態服務
│   ├── app/agent/       # LangGraph：supervisor 多智能體
│   └── docs/API.md
│
├── frontend-old/        # 舊 Vue 前端（僅保留原始碼，不再部署）
├── docs/                # 設計文件
├── design-system/       # 設計 tokens 與準則
├── docker-compose.yml   # 新前端＋後端
└── README.md
```

## 快速開始

### 後端（FastAPI，預設 port 8000）

```bash
cd backend
python -m venv .venv            # 若沒有 .venv
.venv/bin/pip install -r requirements.txt
cp .env.example .env            # 填入 TAVILY_API_KEY / GOOGLE_MAPS_API_KEY / DATABASE_URL / Ollama 網址
.venv/bin/python server.py      # 目錄內有 *.ts.net.crt 時自動開 HTTPS
```

- 健康檢查：`curl -k https://localhost:8000/api/categories`
- 完整 API 文件：`backend/docs/API.md`

### 前端（新前端，預設 port 5173）

```bash
cd frontend
npm ci
npm run dev        # http://localhost:5173
```

開發伺服器會把 `/api` 與 `/uploads` 代理到後端，預設目標是 `https://localhost:8000`。
可在 `frontend/.env` 設定 `API_BASE_URL` 覆寫；完整說明見 [`frontend/README.md`](frontend/README.md)。

### 建置並讓新前端上線

```bash
cd frontend
npm run build      # 產出 frontend/dist
cd ../backend
.venv/bin/python server.py
```

後端 `app/server.py` 會服務 `frontend/dist`（`/` 回 index.html，`/pages/*` 等靜態檔直接回）。
因此建置完成後，開啟 `https://<backend 網址>:8000/` 就是新前端，且 API 同源；這是讓手機 GPS 在所有裝置都能使用的建議方式。

### 一鍵啟動（Docker Compose，選用）

```bash
docker compose up --build
# 前端 http://localhost:5173/ ─ /api、/uploads ─▶ backend:8000
```

> 後端依賴外部 PostgreSQL（pgvector）與 Ollama；請依 `backend/.env` 設定連線。

## HTTPS 與 GPS 定位

`navigator.geolocation` 只在安全 context 下開放：

| 存取方式 | GPS |
|---|---|
| `http://localhost:5173`（本機） | ✅ |
| `https://<tailnet>.ts.net:8000/`（後端直接服務新前端） | ✅ |
| 手機經 LAN IP／純 http 網址 | ❌ |

要讓手機使用 GPS，建議以建置後的前端由後端 HTTPS 同源服務。

## 前端 → 後端 API

| 前端呼叫 | 後端端點 | 說明 |
|---|---|---|
| `getWeather(lat, lng)` | `GET /api/weather` | Open-Meteo 目前天氣 |
| `streamChat()` | `POST /api/chat/stream` | LangGraph Supervisor SSE 串流 |
| `analyze(images)` | `POST /api/ai-analyze` | AI 看照片選類別＋寫描述 |
| `upload(files)` | `POST /api/uploads` | 照片上傳（multipart） |
| `searchPlaces(q)` | `GET /api/places` | 地點搜尋 |
| `reverse(lat, lng)` | `GET /api/places/reverse` | 座標反查地址 |
| `createCase(payload)` | `POST /api/cases` | 建立申報案件 |

前端 API 設定集中於 `frontend/js/api/community-api.js`；建置／開發前會由
`frontend/scripts/generate-runtime-config.mjs` 依 `.env` 產生 `js/api/runtime-config.js`。

## 環境變數（前端）

| 變數 | 預設 | 說明 |
|---|---|---|
| `API_BASE_URL` | `https://localhost:8000` | 後端根網址（可含 `/api`） |
| `API_USE_PROXY` | `true` | `true` 打同源 `/api`；`false` 直連後端（需 CORS） |
| `MOCK_AUTHOR_ACCOUNT` | `test1` | 申報案件使用的作者帳號 |
| `GOOGLE_MAPS_API_KEY` | 空 | 地圖用；留空時地圖降級 |
| `DEV_TUNNEL_HOST` | 空 | 開發隧道主機名稱 |

## 舊前端

`frontend-old/` 僅保留原始碼與 Git 歷史，未列入 Docker Compose，後端也不再指向它。
新前端上線後，舊前端即下線；確認不再需要時可自行移除該目錄。
