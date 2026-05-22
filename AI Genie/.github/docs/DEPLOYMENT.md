# Chatbot Platform — 完整部署說明

> 本文件供交接人員或 AI 開發工具（Claude Code、GitHub Copilot 等）作為上下文使用，涵蓋本機開發環境與伺服器生產部署的完整流程。

---

## 1. 系統需求

### 本機開發
- Next.js（需要 Node.js）
- Python 3.11+
- Redis（本機或 Docker）
- PostgreSQL 16+（本機或 Docker）
- Git

### 伺服器部署
- Docker + Docker Compose
- 至少 2GB RAM
- 開放 port（預設由 `.env` 的 `NGINX_PORT` 決定）

---

## 2. 本機開發環境設定

### 2.1 前端設定

```bash
# 安裝依賴
npm install

# 建立環境變數
cp .env.example .env
```

編輯根目錄 `.env`：
```env
NGINX_PORT=80
POSTGRES_PASSWORD=your_local_password

# 前端環境變數（本機開發用）
NEXT_PUBLIC_API_URL=http://localhost:5000
NEXT_PUBLIC_ADMIN_API_URL=http://localhost:5001
NEXT_PUBLIC_FRONTEND_URL=http://localhost:3000
```

啟動前端開發伺服器：
```bash
npm run dev
# 前端跑在 http://localhost:3000
```

### 2.2 後端設定

```bash
cd backend

# 建立虛擬環境
python -m venv venv

# 啟動虛擬環境
# Windows:
venv\Scripts\activate
# macOS/Linux:
source venv/bin/activate

# 安裝依賴
pip install -r requirements.txt
```

編輯 `backend/.env`：
```env
# Gemini API Key（各租戶獨立，格式：TENANT_{ID大寫}_GEMINI_API_KEY）
TENANT_DEMO_GEMINI_API_KEY=AIzaSy...
TENANT_TOURISM_GEMINI_API_KEY=AIzaSy...

# Admin API Key（管理後台登入用）
ADMIN_API_KEY=admin_secret_key

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0
REDIS_PASSWORD=

# PostgreSQL
POSTGRES_PASSWORD=your_local_password
DATABASE_URL=postgresql://aigenie:your_local_password@localhost:5432/aigenie

# Frappe ERP（如有使用 FrappeQueryService）
FRAPPE_URL=
FRAPPE_API_KEY=
FRAPPE_API_SECRET=
```

### 2.3 啟動本機服務

需要同時啟動以下服務：

```bash
# 1. Redis（如果沒有本機 Redis，用 Docker）
docker run -d --name redis -p 6379:6379 redis:latest

# 2. PostgreSQL（如果沒有本機 PG，用 Docker）
docker run -d --name postgres \
  -e POSTGRES_DB=aigenie \
  -e POSTGRES_USER=aigenie \
  -e POSTGRES_PASSWORD=your_local_password \
  -p 5432:5432 \
  postgres:16-alpine

# 3. Chat API（在 backend/ 目錄）
python app.py
# 跑在 http://localhost:5000

# 4. Admin API（在 backend/ 目錄，另開終端）
python admin_app.py
# 跑在 http://localhost:5001

# 5. 前端（在根目錄，另開終端）
npm run dev
# 跑在 http://localhost:3000
```

### 2.4 本機驗證

- 管理後台：http://localhost:3000/admin/login（預設 API Key: `admin_secret_key`）
- 聊天頁面：http://localhost:3000/chat?tenant_id=demo
- Chat API 健康檢查：http://localhost:5000/api/health
- Admin API 健康檢查：http://localhost:5001/api/admin/health

---

## 3. 伺服器部署（Docker Compose）

### 3.1 準備伺服器

```bash
# 安裝 Docker & Docker Compose
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# 登出再登入讓 docker group 生效
```

### 3.2 上傳專案

```bash
# 方法 1: Git clone
git clone <repo-url> /opt/chatbot-platform
cd /opt/chatbot-platform

# 方法 2: rsync/scp
rsync -avz --exclude node_modules --exclude .next --exclude __pycache__ \
  ./ user@server:/opt/chatbot-platform/
```

### 3.3 設定環境變數

```bash
# 根目錄 .env（Docker Compose 讀取）
cp .env.example .env
```

編輯根目錄 `.env`：
```env
# Nginx 對外 Port
NGINX_PORT=8080

# PostgreSQL 密碼
POSTGRES_PASSWORD=一個強密碼

# 前端環境變數（build time，全部走同一個域名）
# 如果有域名：
NEXT_PUBLIC_API_URL=https://your-domain.com
NEXT_PUBLIC_ADMIN_API_URL=https://your-domain.com
NEXT_PUBLIC_FRONTEND_URL=https://your-domain.com

# 如果用 IP + Port：
# NEXT_PUBLIC_API_URL=http://YOUR_SERVER_IP:8080
# NEXT_PUBLIC_ADMIN_API_URL=http://YOUR_SERVER_IP:8080
# NEXT_PUBLIC_FRONTEND_URL=http://YOUR_SERVER_IP:8080
```

```bash
# 後端 .env
cp backend/.env.example backend/.env
```

編輯 `backend/.env`：
```env
# 各租戶 Gemini API Key
TENANT_DEMO_GEMINI_API_KEY=AIzaSy...
TENANT_TOURISM_GEMINI_API_KEY=AIzaSy...
TENANT_CAESARPARK_GEMINI_API_KEY=AIzaSy...
# ... 其他租戶

# Admin API Key（建議改為強密碼）
ADMIN_API_KEY=your_strong_admin_key

# Redis（Docker 內部網路，由 docker-compose 覆蓋）
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_DB=0
REDIS_PASSWORD=

# PostgreSQL（Docker 內部網路，由 docker-compose 覆蓋）
DATABASE_URL=postgresql://aigenie:一個強密碼@postgres:5432/aigenie

# Frappe ERP（如有使用）
FRAPPE_URL=https://your-erp.example.com
FRAPPE_API_KEY=your_key
FRAPPE_API_SECRET=your_secret
```

### 3.4 準備資料目錄

```bash
# Docker Compose 會掛載這些目錄
mkdir -p backend/data
mkdir -p backend/translations
mkdir -p backend/data/images/tenants

# 複製 tenants.json 到 data 目錄（Docker 掛載用）
cp backend/config/tenants.json backend/data/tenants.json
```

> **重要**：Docker 環境中 `TENANTS_CONFIG_PATH=/app/data/tenants.json`，所以生產環境的設定檔在 `backend/data/tenants.json`，而非 `backend/config/tenants.json`。

### 3.5 建置與啟動

```bash
# 首次建置（會花較長時間）
docker compose build

# 啟動所有服務（背景執行）
docker compose up -d

# 查看日誌
docker compose logs -f

# 查看特定服務日誌
docker compose logs -f backend
docker compose logs -f admin-backend
docker compose logs -f frontend
```

### 3.6 驗證部署

```bash
# 健康檢查
curl http://localhost:8080/api/health
# 預期回應: {"status": "healthy"}

curl -H "X-Admin-Key: your_strong_admin_key" http://localhost:8080/api/admin/health
# 預期回應: {"status": "healthy"}

# 前端
curl -I http://localhost:8080
# 預期回應: HTTP/1.1 200 OK
```

瀏覽器訪問：
- 管理後台：`http://YOUR_SERVER_IP:8080/admin`
- 聊天頁面：`http://YOUR_SERVER_IP:8080/chat?tenant_id=demo`

---

## 4. Docker Compose 服務架構

```
┌─────────────────────────────────────────────────────┐
│                    Nginx (:80)                        │
│  ┌─────────────┬──────────────┬──────────────────┐  │
│  │ /           │ /api/        │ /api/admin/      │  │
│  │ → frontend  │ → backend    │ → admin-backend  │  │
│  │   :3000     │   :5000      │   :5001          │  │
│  └─────────────┴──────────────┴──────────────────┘  │
│  │ /images/tenants/ → 靜態檔案 serve               │  │
└─────────────────────────────────────────────────────┘
         ↕                ↕              ↕
    ┌─────────┐    ┌──────────┐    ┌──────────┐
    │  Redis  │    │ Postgres │    │  Volumes │
    │  :6379  │    │  :5432   │    │ (data)   │
    └─────────┘    └──────────┘    └──────────┘
```

### 各服務說明

| 服務 | 容器名稱 | 說明 |
|------|----------|------|
| redis | ai-genie-redis | Session 快取 |
| postgres | ai-genie-postgres | 訊息記錄、統計 |
| backend | ai-genie-backend | Chat API (gunicorn 4 workers) |
| admin-backend | ai-genie-admin-backend | Admin API (gunicorn 2 workers) |
| frontend | ai-genie-frontend | Next.js SSR |
| nginx | ai-genie-nginx | 反向代理 + 靜態檔案 |

### Volume 掛載

| 主機路徑 | 容器路徑 | 用途 |
|----------|----------|------|
| `backend/.env` | `/app/.env` | 環境變數（含 API Keys） |
| `backend/data/` | `/app/data/` | tenants.json 設定檔 |
| `backend/prompts/` | `/app/prompts/` | 提示詞檔案 |
| `backend/translations/` | `/app/translations/` | 翻譯檔 |
| `backend/data/images/tenants/` | `/app/public/images/tenants/` | 上傳圖片 |

---

## 5. Nginx 路由規則

```nginx
# 租戶圖片（靜態 serve，7 天快取）
location /images/tenants/ {
    alias /usr/share/nginx/images/tenants/;
    expires 7d;
}

# Admin API（最長前綴優先匹配）
location /api/admin/ {
    proxy_pass http://admin-backend:5001;
}

# Chat API（timeout 120s，等待 Gemini 回應）
location /api/ {
    proxy_pass http://backend:5000;
    proxy_read_timeout 120s;
}

# 前端（WebSocket 支援）
location / {
    proxy_pass http://frontend:3000;
}
```

---

## 6. 常用維運操作

### 6.1 更新程式碼

```bash
cd /opt/chatbot-platform

# 拉取最新程式碼
git pull

# 重新建置並重啟
docker compose build
docker compose up -d

# 如果只改了後端（不需重建前端）
docker compose build backend admin-backend
docker compose up -d backend admin-backend

# 如果只改了前端
docker compose build frontend
docker compose up -d frontend
```

### 6.2 更新租戶設定

租戶設定修改後**不需要重啟**（TenantManager 每次請求都會重新讀取）：
```bash
# 直接編輯
vim backend/data/tenants.json

# 或透過 Admin API / 管理後台操作（推薦）
```

### 6.3 新增租戶 API Key

```bash
# 編輯 backend/.env，新增一行
echo "TENANT_NEWTENANT_GEMINI_API_KEY=AIzaSy..." >> backend/.env

# 重啟後端讓 .env 生效
docker compose restart backend admin-backend
```

### 6.4 查看日誌

```bash
# 所有服務
docker compose logs -f --tail=100

# 特定服務
docker compose logs -f backend --tail=50
docker compose logs -f admin-backend --tail=50

# 查看 Nginx access log
docker exec ai-genie-nginx cat /var/log/nginx/access.log
```

### 6.5 資料庫操作

```bash
# 進入 PostgreSQL
docker exec -it ai-genie-postgres psql -U aigenie -d aigenie

# 查看今日訊息量
SELECT tenant_id, COUNT(*) FROM message_logs
WHERE created_at >= CURRENT_DATE GROUP BY tenant_id;

# 查看活躍 session
SELECT * FROM chat_sessions ORDER BY last_seen DESC LIMIT 20;
```

### 6.6 Redis 操作

```bash
# 進入 Redis CLI
docker exec -it ai-genie-redis redis-cli

# 查看所有 session keys
KEYS gemini_session:*

# 清除特定租戶的所有 session
# 注意：Redis 不支援 pattern delete，需用 scan
redis-cli --scan --pattern "gemini_session:demo:*" | xargs redis-cli DEL
```

### 6.7 備份

```bash
# 備份 PostgreSQL
docker exec ai-genie-postgres pg_dump -U aigenie aigenie > backup_$(date +%Y%m%d).sql

# 備份設定檔
cp backend/data/tenants.json backup/tenants_$(date +%Y%m%d).json
cp backend/.env backup/env_$(date +%Y%m%d)
```

### 6.8 完全重置

```bash
# 停止並刪除所有容器和 volume（資料會遺失！）
docker compose down -v

# 重新建置
docker compose build --no-cache
docker compose up -d
```

---

## 7. HTTPS 設定（生產環境建議）

### 方法 1：外部反向代理（推薦）

在 Docker Nginx 前面再加一層主機 Nginx 處理 SSL：

```nginx
# /etc/nginx/sites-available/chatbot
server {
    listen 443 ssl;
    server_name your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;  # Docker Nginx port
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }
}

server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$host$request_uri;
}
```

申請 Let's Encrypt 憑證：
```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com
```

### 方法 2：Cloudflare（最簡單）

1. 將域名 DNS 指向伺服器 IP
2. Cloudflare 設定 SSL 模式為 "Flexible" 或 "Full"
3. 不需要在伺服器上設定 SSL

---

## 8. 環境變數完整清單

### 根目錄 `.env`

| 變數 | 說明 | 範例 |
|------|------|------|
| `NGINX_PORT` | Nginx 對外 port | `8080` |
| `POSTGRES_PASSWORD` | PostgreSQL 密碼 | `strong_password` |
| `NEXT_PUBLIC_API_URL` | 前端呼叫 Chat API 的 URL | `https://your-domain.com` |
| `NEXT_PUBLIC_ADMIN_API_URL` | 前端呼叫 Admin API 的 URL | `https://your-domain.com` |
| `NEXT_PUBLIC_FRONTEND_URL` | 前端自身 URL（widget 用） | `https://your-domain.com` |

### `backend/.env`

| 變數 | 說明 | 範例 |
|------|------|------|
| `TENANT_{ID}_GEMINI_API_KEY` | 各租戶 Gemini API Key | `AIzaSy...` |
| `ADMIN_API_KEY` | 管理後台驗證 Key | `your_admin_key` |
| `REDIS_HOST` | Redis 主機 | `localhost` / `redis` |
| `REDIS_PORT` | Redis Port | `6379` |
| `REDIS_DB` | Redis DB 編號 | `0` |
| `REDIS_PASSWORD` | Redis 密碼 | （空或密碼） |
| `DATABASE_URL` | PostgreSQL 連線字串 | `postgresql://aigenie:pw@host:5432/aigenie` |
| `FRAPPE_URL` | Frappe ERP URL | `https://erp.example.com` |
| `FRAPPE_API_KEY` | Frappe API Key | `...` |
| `FRAPPE_API_SECRET` | Frappe API Secret | `...` |

> **注意**：Docker Compose 中 `REDIS_HOST`、`DATABASE_URL`、`TENANTS_CONFIG_PATH` 會被 `environment` 區塊覆蓋，不需要在 `backend/.env` 中設定 Docker 內部值。

---

## 9. 故障排除

### 前端 build 失敗
```bash
# 清除快取重建
docker compose build --no-cache frontend
```

### 後端連不到 Redis
```bash
# 確認 Redis 容器健康
docker compose ps redis
docker exec ai-genie-redis redis-cli ping
# 預期回應: PONG
```

### 資料庫連線失敗
```bash
# 確認 PostgreSQL 容器健康
docker compose ps postgres
docker exec ai-genie-postgres pg_isready -U aigenie
```

### Gemini API 回應慢或失敗
- 確認 API Key 有效且有額度
- 檢查 `backend` 日誌中的 `⏱️` 時間統計
- `url_context` 失敗會自動 fallback，看日誌中是否有 `⚠️ url_context 抓取失敗`

### 租戶設定不生效
- 確認編輯的是正確的 `tenants.json`（Docker 中是 `backend/data/tenants.json`）
- TenantManager 每次請求都會重新讀取，不需重啟
- 但 `.env` 中的 API Key 修改需要重啟後端容器

### 圖片上傳後看不到
- 確認 Nginx volume 掛載正確：`backend/data/images/tenants:/usr/share/nginx/images/tenants:ro`
- 確認 admin-backend 的 `UPLOAD_IMAGES_PATH` 環境變數設定正確

### 管理後台登入失敗
- 確認 `backend/.env` 中的 `ADMIN_API_KEY` 值
- 確認前端 `NEXT_PUBLIC_ADMIN_API_URL` 指向正確的 admin-backend

---

## 10. 效能調校建議

1. **Gunicorn Workers**：
   - Chat API: 4 workers（CPU 密集型，等待 Gemini 回應）
   - Admin API: 2 workers（低流量）
   - 可根據 CPU 核心數調整：`2 * CPU + 1`

2. **Redis**：
   - Session TTL 預設 1 小時，可根據需求調整
   - 如果記憶體不足，可降低 `max_workers` 或縮短 TTL

3. **PostgreSQL**：
   - 連線池：`pool_size=10, max_overflow=20`
   - 定期清理舊的 `message_logs`（建議超過 90 天的資料歸檔或刪除）

4. **Nginx**：
   - 靜態圖片已設定 7 天快取
   - API timeout 設為 120 秒（Gemini 回應可能較慢）

5. **前端**：
   - Next.js 已使用 standalone output
   - 靜態資源由 Nginx 快取

6. **TenantManager Hot Reload**：
   - 目前每次請求都讀檔，高流量時可改為定時重載（如每 30 秒）
   - 修改 `get_tenant()` 的 `auto_reload` 參數
