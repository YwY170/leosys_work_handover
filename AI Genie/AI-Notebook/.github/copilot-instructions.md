# Copilot Instructions

本專案是 NotebookLM LINE Bot Platform — 將 Google NotebookLM AI 問答整合到 LINE 官方帳號的平台。

## 專案文件

- `ARCHITECTURE.md` — 完整程式架構、資料庫 schema、API 端點、業務流程
- `DEPLOYMENT.md` — 本機開發與伺服器部署完整指南

## 技術棧

- Backend: Python 3.12 + FastAPI + aiosqlite (SQLite)
- Frontend: React 18 + TypeScript + Vite + Tailwind CSS
- 部署: Docker Compose (backend + nginx frontend)
- 外部整合: LINE Messaging API, notebooklm-py

## 程式碼風格

### Python (Backend)
- 非同步函式優先（async/await）
- DB 操作：`async with aiosqlite.connect(DB) as db`
- 型別標註：使用 Python 3.12 語法（`str | None` 而非 `Optional[str]`）
- Router 放 `backend/routers/`，業務邏輯放 `backend/services/`
- Pydantic model 放 `backend/models.py`
- 環境變數透過 `config.py` 的 `Settings` class 管理

### TypeScript (Frontend)
- React 函式元件 + hooks
- API client 統一在 `frontend/src/lib/api.ts`
- 頁面元件在 `frontend/src/pages/`
- Tailwind CSS 樣式，使用專案自訂 class（card-raised, btn-primary, input-tactile）
- 路由使用 react-router-dom v6

## 關鍵設計模式

1. LINE Webhook 問答使用 `asyncio.create_task` 背景處理，回答用 push message（非 reply）
2. NLM 認證資訊（storage_state）使用 Fernet 加密存儲在 SQLite
3. `nlm_service.py` 使用 `asyncio.Lock` 做並發控制
4. 資料庫 migration 在 `database.py` 的 `init_db()` 中自動執行
5. 前端三步驟設定精靈：LINE Channel → NLM 綁定 → 選筆記本

## 安全規範

- NLM cookie 必須經過 Fernet 加密才能存入 DB
- LINE webhook 必須驗證 x-line-signature（HMAC-SHA256）
- 管理員 API 需要 admin_password 參數驗證
- 不在 log 中輸出 token、cookie、密碼等敏感資訊

## 禁止事項

- 不要把 SQLite 換成其他資料庫
- 不要移除 nlm_service.py 中的 asyncio.Lock
- 不要用 reply message 回覆 NLM 查詢結果（LINE 5 秒 timeout）
- 不要修改 ENCRYPTION_KEY 除非準備重新綁定所有帳號
