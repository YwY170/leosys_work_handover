# NotebookLM LINE Bot Platform — 完整程式架構說明

> 本文件供 AI 開發工具（Claude Code、GitHub Copilot CLI 等）作為專案上下文，以便接續開發。
> 使用方式：在 Claude Code 中用 `/read ARCHITECTURE.md` 或對話中提及此檔案；在 Copilot CLI 中將此檔案內容作為 context 提供。

---

## 專案概述

這是一個將 Google NotebookLM 的 AI 問答能力整合到 LINE 官方帳號的平台。

**核心使用場景**：講師透過管理介面批量建立學員帳號 → 學員透過邀請碼自助設定 LINE Channel 與 NotebookLM 綁定 → 學員在 LINE 上提問即可獲得 NotebookLM 的 AI 回答。

---

## 技術棧

| 層級 | 技術 |
|------|------|
| Backend | Python 3.12 + FastAPI + Uvicorn |
| Database | SQLite（aiosqlite 非同步存取） |
| Frontend | React 18 + TypeScript + Vite + Tailwind CSS |
| 部署 | Docker Compose（backend + nginx frontend） |
| 外部整合 | LINE Messaging API、Google NotebookLM（notebooklm-py） |

---

## 目錄結構

```
notebooklm-line/
├── backend/                  # FastAPI 後端
│   ├── main.py              # 應用程式入口、lifespan、CORS、路由掛載
│   ├── config.py            # Pydantic Settings 環境變數管理
│   ├── database.py          # SQLite 初始化與自動 migration
│   ├── models.py            # Pydantic request/response models
│   ├── routers/
│   │   ├── webhook.py       # LINE Webhook 接收與處理
│   │   ├── admin.py         # 管理員 API（邀請碼、學員管理、CSV 匯入匯出）
│   │   └── auth.py          # NotebookLM 綁定與筆記本選擇
│   ├── services/
│   │   ├── line_service.py  # LINE Messaging API 封裝（reply/push/loading）
│   │   ├── nlm_service.py   # NotebookLM 操作封裝（綁定/列表/問答）
│   │   ├── crypto_service.py # Fernet 對稱加解密
│   │   └── text_formatter.py # Markdown → LINE 純文字轉換 + 引用來源解析
│   ├── data/                # SQLite 資料庫檔案（volume 掛載）
│   ├── test/                # pytest 測試
│   ├── requirements.txt     # Python 依賴
│   └── Dockerfile
├── frontend/                 # React SPA
│   ├── src/
│   │   ├── App.tsx          # 路由定義（/, /setup, /admin）
│   │   ├── pages/
│   │   │   ├── InvitePage.tsx   # 邀請碼驗證頁
│   │   │   ├── SetupPage.tsx    # 三步驟設定精靈（LINE Channel → NLM 綁定 → 選筆記本）
│   │   │   └── AdminPage.tsx    # 管理員後台
│   │   └── lib/api.ts      # 前端 API client 封裝
│   ├── nginx.conf           # Nginx 反向代理（/api → backend, /webhook → backend）
│   ├── vite.config.ts       # 開發代理設定
│   ├── package.json
│   └── Dockerfile           # 多階段建置（node build → nginx serve）
├── login-tool/              # 桌面端 NLM 登入輔助工具（Playwright）
│   ├── login.py             # 開啟瀏覽器讓使用者登入 Google，擷取 storage_state.json
│   └── python/              # 內嵌 Python 環境（Windows 發行用）
└── docker-compose.yml       # 容器編排（backend + frontend）
```

---

## 資料庫 Schema（SQLite）

### channels 表
```sql
CREATE TABLE channels (
    channel_id TEXT PRIMARY KEY,           -- LINE Channel ID
    channel_secret TEXT NOT NULL,          -- LINE Channel Secret
    channel_access_token TEXT NOT NULL,    -- LINE Channel Access Token
    nlm_auth_json_encrypted TEXT,          -- Fernet 加密的 NotebookLM storage_state
    notebook_id TEXT,                      -- 選定的筆記本 ID
    expires_at DATETIME,                   -- 到期時間（課程結束日）
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### invite_codes 表
```sql
CREATE TABLE invite_codes (
    code TEXT PRIMARY KEY,                 -- 邀請碼（token_urlsafe(8)）
    student_name TEXT DEFAULT '',          -- 學員姓名
    used INTEGER DEFAULT 0,               -- 是否已使用
    channel_id TEXT,                       -- 關聯的 Channel ID
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## API 端點一覽

### 學員端（需 session token）
| Method | Path | 說明 |
|--------|------|------|
| POST | `/api/verify-invite` | 驗證邀請碼，回傳 session token |
| POST | `/api/channels?token=xxx` | 建立/更新 LINE Channel |
| GET | `/api/channels/{id}?token=xxx` | 取得 Channel 資訊 |
| POST | `/api/channels/{id}/nlm-login` | 上傳 storage_state.json 綁定 NLM |
| POST | `/api/channels/{id}/nlm-bind-local` | 從伺服器本機讀取 storage_state |
| GET | `/api/channels/{id}/nlm-status` | 查詢 NLM 綁定狀態 |
| GET | `/api/channels/{id}/notebooks` | 列出可用筆記本 |
| PUT | `/api/channels/{id}/notebook` | 選擇使用的筆記本 |

### 管理員端（需 admin_password）
| Method | Path | 說明 |
|--------|------|------|
| POST | `/api/invite-codes/generate` | 批量產生邀請碼 |
| POST | `/api/admin/import-csv` | CSV 匯入學員名單 |
| GET | `/api/admin/students` | 查看所有學員狀態 |
| PUT | `/api/admin/set-expiry` | 設定全域到期日 |
| DELETE | `/api/admin/channels/{id}` | 刪除 Channel |
| DELETE | `/api/admin/clear-all` | 清除所有綁定 |
| GET | `/api/admin/export-csv` | 匯出學員對照表 CSV |

### Webhook
| Method | Path | 說明 |
|--------|------|------|
| POST | `/webhook/{channel_id}` | LINE Webhook 接收端點 |

---

## 核心業務流程

### 1. LINE Webhook 問答流程（最重要）

```
LINE 用戶發送文字訊息
  → POST /webhook/{channel_id}
  → 從 DB 查詢 channel 資訊
  → HMAC-SHA256 驗證 LINE 簽章
  → 檢查 channel 是否過期
  → 嘗試顯示 loading animation（LINE API）
  → asyncio.create_task 背景處理：
      → 解密 storage_state
      → 呼叫 NotebookLM chat.ask()
      → 格式化回答（Markdown → 純文字）
      → 解析引用來源 [1][2]...
      → push message 回答給用戶
      → push message 引用來源（如有）
```

**關鍵設計決策**：
- 使用背景 task 避免 LINE 5 秒 webhook timeout
- 回答用 push message（非 reply），因為背景處理時 reply token 已過期
- 長文自動切割為 5000 字元 chunks（LINE 單則訊息限制），最多 5 則
- NLM 並發控制：`asyncio.Lock` 防止多請求同時修改環境變數

### 2. 學員自助設定流程

```
輸入邀請碼 → 驗證通過取得 token → 輸入 LINE Channel 資訊 → 取得 Webhook URL
→ 上傳 storage_state.json → NLM 綁定成功 → 選擇筆記本 → 完成
```

### 3. 到期自動清理

- `main.py` 的 lifespan 啟動背景排程 `expiry_scheduler()`
- 每 60 秒檢查 `channels.expires_at`
- 過期 channel 自動刪除，關聯 invite_codes 也一併刪除

---

## 安全機制

| 機制 | 實作方式 |
|------|----------|
| NLM cookie 加密 | Fernet 對稱加密（`ENCRYPTION_KEY` 環境變數） |
| 學員認證 | 邀請碼驗證 → in-memory session token（`secrets.token_urlsafe(32)`） |
| 管理員認證 | 密碼比對（`ADMIN_PASSWORD` 環境變數） |
| LINE Webhook 驗證 | HMAC-SHA256 簽章驗證（channel_secret） |
| 到期控制 | 背景排程自動清理過期 channel |

---

## 環境變數

```env
# 必填
ENCRYPTION_KEY=    # Fernet 金鑰（python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"）
WEBHOOK_BASE_URL=  # 對外 HTTPS URL（如 https://your-domain.com）
ADMIN_PASSWORD=    # 管理員密碼

# 選填
DB_PATH=data/data.db  # SQLite 路徑（預設 data.db）
APP_PORT=8083          # Docker 對外 port（docker-compose.yml 使用）
```

---

## 外部依賴說明

### notebooklm-py（非官方 SDK）
- 透過 Google cookie（storage_state.json）認證
- 主要方法：`notebooks.list()`、`chat.ask(notebook_id, question)`、`sources.list(notebook_id)`
- 程式內動態設定 `NOTEBOOKLM_AUTH_JSON` 環境變數
- 有並發問題，需用 Lock 保護

### LINE Messaging API
- **Reply message**：5 秒內回覆，用於即時回應
- **Push message**：主動推送，用於非同步回答（本專案主要使用）
- **Loading animation**：顯示打字中動畫（`/v2/bot/chat/loading`）
- **Webhook signature**：`x-line-signature` header，HMAC-SHA256 驗證

---

## 開發注意事項

1. **NLM 並發控制**：`nlm_service.py` 使用 `asyncio.Lock` 防止多個請求同時修改 `NOTEBOOKLM_AUTH_JSON` 環境變數，這是因為 notebooklm-py 從環境變數讀取認證資訊
2. **資料庫 migration**：`database.py` 的 `init_db()` 包含自動 migration（PRAGMA table_info 檢查後 ALTER TABLE），新增欄位時請遵循此模式
3. **文字格式化**：`text_formatter.py` 將 NLM 回傳的 Markdown 轉為 LINE 友善純文字（移除 #、**、* 等標記）
4. **引用來源**：回答中的 `[1]`、`[2]` 引用會被解析，附上來源標題作為獨立第二則訊息
5. **Session 管理**：學員 session 為 in-memory dict（`_sessions`），伺服器重啟後失效，設計上可接受因為設定流程短暫
6. **前端開發代理**：`vite.config.ts` 將 `/api` 和 `/webhook` 代理到 `http://127.0.0.1:8000`
7. **靜態檔案 fallback**：`main.py` 會嘗試掛載 `static/` 或 `../frontend/dist/` 作為 SPA，Docker 部署時由 Nginx 處理所以不會觸發
8. **CSV 匯入**：支援多種姓名欄位名稱（name、姓名、學員姓名），支援頓號/逗號/斜線分隔多人，自動移除括號註記
