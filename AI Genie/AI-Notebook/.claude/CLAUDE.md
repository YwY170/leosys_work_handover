# CLAUDE.md

本專案是 NotebookLM LINE Bot Platform — 將 Google NotebookLM AI 問答整合到 LINE 官方帳號的平台。

## 專案文件

- `ARCHITECTURE.md` — 完整程式架構、資料庫 schema、API 端點、業務流程
- `DEPLOYMENT.md` — 本機開發與伺服器部署完整指南

開始任何開發工作前，請先閱讀 `ARCHITECTURE.md` 了解專案全貌。

## 技術棧

- Backend: Python 3.12 + FastAPI + aiosqlite (SQLite)
- Frontend: React 18 + TypeScript + Vite + Tailwind CSS
- 部署: Docker Compose (backend + nginx frontend)
- 外部整合: LINE Messaging API, notebooklm-py

## 常用指令

```bash
# 後端開發
cd backend
pip install -r requirements.txt
uvicorn main:app --reload --port 8000

# 前端開發
cd frontend
npm install
npm run dev

# 測試
cd backend
pip install -r requirements-test.txt
pytest test/ -v

# Docker 部署
docker compose up -d --build
docker compose logs -f
```

## 開發規範

### 後端
- 非同步優先：所有 DB 操作使用 `async with aiosqlite.connect(DB)` 
- 新增 API 路由放在 `backend/routers/` 對應檔案
- 新增資料庫欄位：在 `database.py` 的 `init_db()` 加入 migration（PRAGMA table_info 檢查 + ALTER TABLE）
- 環境變數：在 `config.py` 的 `Settings` class 新增，同步更新 `.env.example`
- 業務邏輯封裝在 `backend/services/`，router 只做請求處理

### 前端
- 頁面元件放 `frontend/src/pages/`
- API 呼叫統一放 `frontend/src/lib/api.ts`
- 路由在 `App.tsx` 註冊
- 使用 Tailwind CSS，遵循現有的 class 命名風格（card-raised, btn-primary, input-tactile 等）

### 安全注意
- NLM cookie 必須加密存儲（使用 `services/crypto_service.py`）
- LINE webhook 必須驗證 HMAC-SHA256 簽章
- 不要在 log 中輸出敏感資訊（token、cookie、密碼）
- `ENCRYPTION_KEY` 變更會導致所有已加密資料無法解密

### 不要做的事
- 不要把 SQLite 換成其他資料庫（專案設計為輕量單機部署）
- 不要移除 `asyncio.Lock` 在 nlm_service.py 中的並發控制
- 不要用 reply message 回覆 NLM 查詢結果（會超時），必須用 push message
- 不要修改 `ENCRYPTION_KEY` 除非準備重新綁定所有 NLM 帳號
