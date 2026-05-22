# NotebookLM LINE Bot Platform — 完整部署說明

> 本文件供 AI 開發工具（Claude Code、GitHub Copilot CLI 等）作為部署指引。
> 使用方式：在 Claude Code 中用 `/read DEPLOYMENT.md`；在 Copilot CLI 中將此檔案作為 context。

---

## 前置需求

| 工具 | 版本 | 用途 |
|------|------|------|
| Python | 3.12+ | 後端執行 |
| Node.js | 20+ | 前端建置 |
| Docker & Docker Compose | 最新版 | 容器化部署 |
| LINE Developers 帳號 | — | 建立 Messaging API Channel |
| Google 帳號 | — | NotebookLM 存取 |

---

## 一、本機開發部署

### 1.1 後端啟動

```bash
cd backend

# 建立虛擬環境
python -m venv venv
# Windows
venv\Scripts\activate
# macOS/Linux
source venv/bin/activate

# 安裝依賴
pip install -r requirements.txt

# 建立環境變數檔
cp .env.example .env
```

編輯 `backend/.env`：
```env
ENCRYPTION_KEY=<執行下方指令產生>
WEBHOOK_BASE_URL=https://your-ngrok-domain.ngrok-free.app
DB_PATH=data/data.db
ADMIN_PASSWORD=your-admin-password
```

產生 ENCRYPTION_KEY：
```bash
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

啟動後端：
```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

後端 API 文件：`http://localhost:8000/docs`（FastAPI 自動產生 Swagger UI）

### 1.2 前端啟動

```bash
cd frontend

# 安裝依賴
npm install

# 開發模式（自動代理 /api 和 /webhook 到 localhost:8000）
npm run dev
```

前端開發伺服器：`http://localhost:5173`

Vite 已設定代理規則（`vite.config.ts`）：
- `/api` → `http://127.0.0.1:8000`
- `/webhook` → `http://127.0.0.1:8000`

### 1.3 本機 Webhook 測試

LINE Webhook 需要 HTTPS 公開 URL，本機開發使用 ngrok：

```bash
ngrok http 8000
```

取得 ngrok URL（如 `https://xxxx.ngrok-free.app`），更新 `backend/.env`：
```env
WEBHOOK_BASE_URL=https://xxxx.ngrok-free.app
```

重啟後端使設定生效。

> **注意**：ngrok 免費版每次重啟 URL 會變，需同步更新 LINE Developers Console 的 Webhook URL。

### 1.4 NotebookLM 登入工具

學員需要取得 Google NotebookLM 的 cookie（storage_state.json）：

**方法一：login-tool（Windows 桌面工具，給學員用）**
```bash
cd login-tool
python login.py
# 開啟瀏覽器 → 使用者登入 Google → 看到 NLM 首頁後按 Enter
# 輸出：~/Desktop/storage_state.json
```

**方法二：notebooklm-py CLI（伺服器端，管理員用）**
```bash
pip install notebooklm-py
notebooklm login
# 登入後 storage_state.json 存在 ~/.notebooklm/profiles/ 下
# 可透過 API POST /api/channels/{id}/nlm-bind-local 自動讀取
```

### 1.5 執行測試

```bash
cd backend
pip install -r requirements-test.txt
pytest test/ -v
```

---

## 二、伺服器部署（Docker Compose）

### 2.1 伺服器環境準備

需要：
- Docker Engine 20.10+
- Docker Compose v2+
- 域名 DNS 指向伺服器 IP
- HTTPS 設定（Nginx + Let's Encrypt 或 Cloudflare）

### 2.2 部署步驟

```bash
# 1. Clone 專案
git clone <repo-url> notebooklm-line
cd notebooklm-line

# 2. 建立環境變數
cp backend/.env.example backend/.env
# 編輯 backend/.env（見下方）

# 3. 建置並啟動
docker compose up -d --build

# 4. 確認狀態
docker compose ps
docker compose logs -f
```

`backend/.env` 正式環境設定：
```env
ENCRYPTION_KEY=<Fernet key>
WEBHOOK_BASE_URL=https://your-domain.com
DB_PATH=data/data.db
ADMIN_PASSWORD=<強密碼>
```

### 2.3 Docker Compose 架構說明

```
┌─────────────────────────────────────────────┐
│  Host (port 8083)                           │
│  ┌───────────────────────────────────────┐  │
│  │  frontend (Nginx, port 80)            │  │
│  │  ├── / → React SPA 靜態檔案           │  │
│  │  ├── /api/* → proxy to backend:8000   │  │
│  │  └── /webhook/* → proxy to backend    │  │
│  └───────────────────────────────────────┘  │
│                      │                       │
│  ┌───────────────────────────────────────┐  │
│  │  backend (Uvicorn, port 8000)         │  │
│  │  └── volume: ./data → /app/data       │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

- **frontend container**：Nginx 提供 React SPA + 反向代理 API/Webhook
- **backend container**：FastAPI + Uvicorn，僅內部暴露 port 8000
- **資料持久化**：`./data` 目錄掛載到 backend 的 `/app/data`
- **對外 port**：預設 `8083`，可用 `APP_PORT` 環境變數覆蓋

### 2.4 自訂對外 Port

```bash
APP_PORT=9090 docker compose up -d
```

### 2.5 HTTPS 設定（正式環境必要）

LINE Webhook 強制要求 HTTPS。在 Docker 前方加 Nginx reverse proxy：

```nginx
# /etc/nginx/sites-available/notebooklm-line
server {
    listen 443 ssl;
    server_name your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8083;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$host$request_uri;
}
```

取得 SSL 憑證：
```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com
```

### 2.6 資料備份

SQLite 資料庫位於 host 端 `./data/data.db`：

```bash
# 手動備份
cp ./data/data.db ./data/data.db.backup.$(date +%Y%m%d)

# 定時備份（crontab，每天凌晨 3 點）
0 3 * * * cp /path/to/notebooklm-line/data/data.db /path/to/backup/data.db.$(date +\%Y\%m\%d)
```

### 2.7 更新部署

```bash
cd notebooklm-line
git pull
docker compose up -d --build
docker compose logs -f --tail=50
```

### 2.8 常用維運指令

```bash
# 查看即時 log
docker compose logs -f backend
docker compose logs -f frontend

# 重啟單一服務
docker compose restart backend

# 進入 backend container 除錯
docker compose exec backend bash

# 查看資料庫統計
docker compose exec backend python -c "
import sqlite3
conn = sqlite3.connect('data/data.db')
print('Channels:', conn.execute('SELECT count(*) FROM channels').fetchone()[0])
print('Invite codes:', conn.execute('SELECT count(*) FROM invite_codes').fetchone()[0])
conn.close()
"

# 完全停止
docker compose down

# 停止並刪除 volumes（⚠️ 會刪除資料庫）
docker compose down -v
```

---

## 三、LINE Developers Console 設定

### 3.1 建立 Messaging API Channel

1. 前往 [LINE Developers Console](https://developers.line.biz/)
2. 建立 Provider → 建立 **Messaging API** Channel
3. 記錄：
   - **Channel ID**（Basic settings）
   - **Channel Secret**（Basic settings）
   - **Channel Access Token**（Messaging API 頁面，點 "Issue" 產生 long-lived token）

### 3.2 設定 Webhook

在 Messaging API 頁面：
- **Webhook URL**：`https://your-domain.com/webhook/{channel_id}`
- ✅ 開啟 "Use webhook"
- ❌ 關閉 "Auto-reply messages"
- ❌ 關閉 "Greeting messages"

### 3.3 驗證

點擊 "Verify" 按鈕確認連線成功。

---

## 四、完整操作流程（端到端）

### 管理員初始設定
1. 部署服務（本機或伺服器）
2. 進入 `/admin` 頁面，輸入管理員密碼
3. 匯入學員 CSV 或手動產生邀請碼
4. 將邀請碼分發給學員

### 學員設定流程
1. 開啟平台首頁，輸入邀請碼
2. 在 LINE Developers 建立 Messaging API Channel
3. 將 Channel ID、Secret、Access Token 填入平台
4. 複製 Webhook URL 貼到 LINE Developers Console
5. 使用 login-tool 取得 storage_state.json 並上傳
6. 選擇要使用的 NotebookLM 筆記本
7. 完成！學員的 LINE 官方帳號現在可以回答問題了

---

## 五、故障排除

| 問題 | 原因 | 解決方式 |
|------|------|----------|
| LINE Webhook 驗證失敗 | HTTPS 未設定或 URL 錯誤 | 確認 `WEBHOOK_BASE_URL` 與實際域名一致，確認 HTTPS 憑證有效 |
| NotebookLM 綁定失敗 | cookie 過期或格式錯誤 | 重新執行 login tool 取得新的 storage_state.json |
| 回答超時或無回應 | NLM API 回應慢或帳號被限流 | 檢查 `docker compose logs backend`，確認 NLM 帳號狀態 |
| 前端 API 404 | Nginx 代理未生效 | 確認 `docker compose ps` 兩個容器都在運行 |
| 加密錯誤（InvalidToken） | ENCRYPTION_KEY 變更 | 金鑰變更後舊資料無法解密，需重新綁定所有 NLM |
| 資料庫鎖定 | 多個 process 同時寫入 | 確保只有一個 backend instance 運行 |
| ngrok URL 變更後 webhook 失效 | ngrok 重啟產生新 URL | 更新 .env 的 WEBHOOK_BASE_URL 並重啟後端 |

---

## 六、環境變數完整參考

| 變數名 | 必填 | 預設值 | 說明 |
|--------|------|--------|------|
| `ENCRYPTION_KEY` | ✅ | （空） | Fernet 對稱加密金鑰，保護 NLM cookie |
| `WEBHOOK_BASE_URL` | ✅ | `https://your-domain.com` | 對外 HTTPS URL，用於產生 webhook URL |
| `DB_PATH` | ❌ | `data.db` | SQLite 檔案路徑（Docker 建議 `data/data.db`） |
| `ADMIN_PASSWORD` | ✅ | `changeme` | 管理員 API 密碼 |
| `APP_PORT` | ❌ | `8083` | Docker Compose 對外 port |

---

## 七、給 AI 開發工具的提示

當你需要修改此專案時，請注意：

1. **後端新增 API**：在 `backend/routers/` 下對應檔案新增路由，遵循現有的 aiosqlite 非同步模式
2. **新增資料庫欄位**：在 `database.py` 的 `init_db()` 中加入 migration 邏輯（PRAGMA table_info 檢查 + ALTER TABLE）
3. **前端新增頁面**：在 `frontend/src/pages/` 建立元件，在 `App.tsx` 加入路由，API 呼叫放 `lib/api.ts`
4. **環境變數**：在 `config.py` 的 `Settings` class 新增欄位，同步更新 `.env.example`
5. **Docker 重建**：修改 Dockerfile 或依賴後需 `docker compose up -d --build`
6. **測試**：測試檔案在 `backend/test/`，使用 pytest，fixture 在 `conftest.py`
