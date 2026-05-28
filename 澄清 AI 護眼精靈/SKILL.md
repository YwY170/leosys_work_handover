---
name: cceye-linebot
description: 澄清眼科 AI 護眼精靈 LINE Bot 專案開發指南。當開發、修改、除錯、部署此專案時使用。涵蓋架構理解、程式碼慣例、服務互動、部署流程。任何涉及此 LINE Bot 的眼科 AI 對話、Gemini 整合、社區巡迴檢查、LIFF 綁定、智慧分流等功能開發時，都應使用此 skill。
---

# 澄清眼科 AI 護眼精靈 LINE Bot — 專案 Skill

## 專案概述

澄清眼科智能客服系統，提供眼科 AI 諮詢、智慧分流、社區巡迴檢查管理、異常通知等功能。

- **技術棧**：Python 3.12 / Flask 3.0 / Gunicorn / Redis 7 / PostgreSQL / Google Gemini 2.5 Flash
- **部署環境**：Ubuntu 22.04 內部虛擬機（IP: 192.168.10.214），Systemd 管理
- **LINE SDK**：line-bot-sdk v3（v3 webhooks API，非舊版）

---

## 架構與請求流程

```
LINE 用戶 → Nginx (443) → Gunicorn (8 workers, :5000) → Flask App
                                                           ├─ Redis (session/對話歷史)
                                                           ├─ Gemini 2.5 Flash (AI 對話)
                                                           └─ PostgreSQL (用戶/檢查資料)
```

### 環境連線

| 項目 | 值 |
|------|------|
| 虛擬機 SSH | `ssh ubuntu@192.168.10.214`（密碼：Cceye@dmin07） |
| 部署路徑 | `/home/ubuntu/ChengChing-AI-EyeCare/` |
| PostgreSQL | `localhost:5432`，DB: `eyecare`，User: `eyecare_user`，密碼: `Cceye@dmin07` |
| Redis | `localhost:6379`，DB: 0，無密碼 |

### 核心模組

| 模組 | 路徑 | 職責 |
|------|------|------|
| Flask App Factory | `app/main.py` | `create_app()` 建立 Flask 實例，註冊 blueprints |
| Webhook 路由 | `app/routes/multi_webhook.py` | LINE webhook 入口，支援 `/webhook/<bot_id>` 和 `/webhook/multi` |
| LINE Bot 核心 | `app/services/multi_linebot.py` | 多 bot 管理、訊息分派、Gemini 呼叫 |
| Gemini 服務 | `app/services/gemini_service.py` | AI 對話核心，含 Google Search Grounding、Redis session、安全過濾 |
| 智慧分流 | `app/services/triage_service.py` | 繼承 GeminiService，永遠啟用搜尋，症狀評估 |
| 醫師推薦 | `app/services/doctor_service.py` | 從 `data/doctors.json` 載入，按地區/職稱排序注入 prompt |
| Token 追蹤 | `app/services/token_tracker.py` | Gemini API token 用量與 Grounding 配額追蹤 |
| 健檢資料查詢 | `app/services/health_check_service.py` | Google Sheets 整合，查詢用戶最新健檢紀錄 |
| 健檢 AI 摘要 | `app/services/health_summary_service.py` | 用 Gemini 生成健檢報告摘要 |
| HIS 預約查詢 | `app/services/his_service.py` | 介接耀聖 HIS 系統，查詢掛號/預約資訊 |
| 社區檢查匯入 | `app/services/checkup_import_service.py` | XLS 檔案匯入社區巡迴檢查資料 |
| 異常匯入 | `app/services/abnormal_import_service.py` | XLSX 異常標記匯入 |
| 異常通知 | `app/services/checkup_notify_service.py` | 比對已綁定用戶，LINE Push 異常通知 |
| 資料清洗 | `app/services/data_cleaner.py` | 匯入前資料驗證與清洗 |
| Excel 讀取層 | `app/services/workbook_reader.py` | XLS/XLSX 統一讀取抽象層（xlrd + openpyxl） |
| 就診提醒 | `app/services/appointment_reminder.py` | APScheduler 排程，每日 09:30 推送 |
| 資料庫 | `app/models/database.py` | PostgreSQL 連線池 (ThreadedConnectionPool)，DDL + migration |
| 用戶模型 | `app/models/user.py` + `user_storage.py` | LINE 用戶綁定（身分證 + 手機） |
| LIFF 路由 | `app/routes/liff_routes.py` | 用戶綁定前端頁面 + API |
| 檢查管理 | `app/routes/checkup_routes.py` | 管理後台（登入、上傳、查詢、通知） |
| 路徑管理 | `app/utils/paths.py` | 所有路徑集中管理，禁止硬編碼 |
| Bot 配置 | `app/config/bot_config.py` | 從 `bot_configs.json` + 環境變數載入多 bot 設定 |

---

## 訊息分派流程（`multi_linebot.py`）

用戶發送文字訊息後的處理順序：

```
1. 檢查用戶是否已綁定（UserStorage）
   └─ 未綁定 → 推送 LIFF 綁定按鈕（ButtonsTemplate + LIFF_ID）
   
2. 檢查是否為「健檢摘要」請求（關鍵字：健檢報告/健檢摘要/我的報告/檢查報告）
   └─ 是 → HealthCheckService (Google Sheets) → HealthSummaryService (AI 摘要)
   
3. AI 判斷是否為症狀描述（_is_triage_trigger，用 Gemini 判斷 YES/NO）
   └─ 是 → TriageService + DoctorService（動態注入醫師推薦）
   
4. 其他 → GeminiService 通用對話（cceye_system_prompt）
```

所有回應都會：
- 經過 `_clean_markdown()` 清理 markdown 格式
- 前後加上 `AI_DISCLAIMER` 免責聲明
- 自動判斷是否補充澄清眼科官網連結

---

## 程式碼慣例與規則

### 必須遵守

1. **路徑管理**：所有檔案路徑必須透過 `app/utils/paths.py` 取得，禁止硬編碼絕對路徑
2. **環境變數**：敏感資訊（API key、token、密碼）一律放 `.env`，透過 `python-dotenv` 載入
3. **Bot 配置**：多 bot 設定在 `app/config/bot_configs.json`，私密值從環境變數讀取（`_env` 後綴欄位）
4. **Redis Session**：對話歷史存 Redis，key 格式 `gemini_session:{bot_name}:{user_id}`，TTL 1 小時
5. **日誌格式**：`%(asctime)s - %(name)s - %(levelname)s - %(message)s`，使用 RotatingFileHandler (10MB/5 backups)
6. **LINE SDK v3**：使用 `linebot.v3` 命名空間，不要用舊版 `linebot`
7. **資料庫**：使用 `with get_db() as conn:` context manager，自動 commit/rollback/歸還連線
8. **Blueprint 註冊**：新路由建立 Blueprint 後在 `app/main.py` 的 `create_app()` 中註冊

### System Prompt 管理

- 存放於 `app/config/prompts/` 目錄
- `cceye_system_prompt.txt`：主要 AI 對話 prompt
- `triage_system_prompt.txt`：分流專用 prompt
- `security_prompt.txt`：安全防護 prompt（自動合併到所有 bot 的 system prompt 最前面）

### Gemini 服務設計模式

```python
# 新增 bot 場景時，繼承 GeminiService
class NewService(GeminiService):
    def __init__(self, api_key):
        super().__init__(
            api_key=api_key,
            system_prompt_path=str(YOUR_PROMPT_PATH),
            bot_name="你的服務名稱"
        )
    
    # 覆寫搜尋判斷邏輯
    def _should_use_search(self, question: str) -> bool:
        return True  # 或自訂邏輯
```

### 資料庫 Migration 模式

在 `app/models/database.py` 的 `init_db()` 中：
- DDL 放 `CREATE TABLE IF NOT EXISTS`（冪等）
- Migration 用 `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`（冪等）
- 啟動時自動執行，無需額外 migration 工具

---

## 環境變數（必要）

```env
# LINE Bot
CCEYE_LINE_CHANNEL_ACCESS_TOKEN=
CCEYE_LINE_CHANNEL_SECRET=

# Gemini
CCEYE_GEMINI_API_KEY=

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0

# PostgreSQL
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=eyecare
POSTGRES_USER=eyecare_user
POSTGRES_PASSWORD=Cceye@dmin07

# Flask
DEBUG=false
PORT=5000
FLASK_SECRET_KEY=

# Google Sheets（健檢摘要功能）
GOOGLE_SHEET_ID=
GOOGLE_CREDENTIALS_PATH=config/google_credentials.json

# HIS 系統介接
HIS_BASE_URL=http://127.0.0.1:3000

# LIFF（用戶綁定頁面）
LIFF_ID=
```

---

## 開發流程

### 本地啟動

```bash
source .venv/bin/activate
# 確保 Redis 和 PostgreSQL 已啟動
python3 run.py  # 開發模式，單 worker
```

### 生產部署

```bash
ssh ubuntu@192.168.10.214
cd /home/ubuntu/ChengChing-AI-EyeCare
git pull origin main
source .venv/bin/activate
uv pip install -r requirements.txt
sudo systemctl restart cceye-linebot.service
curl http://localhost:5000/health  # 驗證
```

### Gunicorn 配置重點 (`config/gunicorn.conf.py`)

- 8 workers (sync)，bind `0.0.0.0:5000`
- `max_requests=1000`（防記憶體洩漏）
- `timeout=120`（Gemini 回應可能較慢）
- `preload_app=True`
- `post_fork` hook：只在 worker#1 啟動 APScheduler（避免重複排程）

---

## 功能規格文件

完整功能規格在 `specs/` 目錄：

| 編號 | 模組 | 狀態 |
|------|------|------|
| 001 | 護眼運動 Web App | 規劃中 |
| 002 | LINE Bot RAG 衛教知識庫 | 規劃中 |
| 003 | 智慧分流 | **已實作** |
| 004 | 醫囑提醒 | 部分實作（就診提醒） |
| 005 | 預約掛號 | 規劃中 |

---

## 新增功能 Checklist

新增一個 LINE Bot 功能時：

1. [ ] 在 `app/config/prompts/` 建立對應的 system prompt
2. [ ] 在 `app/services/` 建立服務（繼承 `GeminiService` 或獨立）
3. [ ] 在 `app/services/multi_linebot.py` 中整合訊息分派邏輯（注意分派順序）
4. [ ] 如需新資料表，在 `app/models/database.py` 的 `init_db()` 加 DDL
5. [ ] 如需新路由，建立 Blueprint 並在 `app/main.py` 註冊
6. [ ] 如需新路徑，在 `app/utils/paths.py` 定義
7. [ ] 更新 `requirements.txt`（如有新依賴）
8. [ ] 如需新環境變數，更新 `.env` 並通知部署人員
9. [ ] 測試：`curl -X POST http://localhost:5000/webhook/cceye` 驗證 webhook
10. [ ] 如修改安全相關邏輯，跑 `redteam/run_redteam.sh` 回歸測試

---

## 安全與紅隊測試

### 安全架構

本專案有三層 AI 安全防護：

1. **Security Prompt**（`app/config/prompts/security_prompt.txt`）— 自動合併到所有 bot 的 system prompt 最前面
   - 禁止洩漏 system prompt
   - 禁止角色切換
   - Anti-hallucination 規則
   - 緊急狀況處理原則

2. **輸入預過濾**（`gemini_service.py` 的 `_pre_filter_input()`）— 第一層攔截惡意輸入

3. **可疑輸入標記**（`_is_suspicious_input()`）— 標記後觸發輸出審查

### 紅隊測試（`redteam/` 目錄）

使用 [Promptfoo](https://www.promptfoo.dev/) 框架進行自動化紅隊攻擊測試：

| 檔案 | 用途 |
|------|------|
| `redteam/promptfooconfig.yaml` | 手動測試案例（指令覆蓋、DAN、情感操縱、prompt 竊取等） |
| `redteam/redteam_auto.yaml` | 自動生成攻擊（prompt-extraction、hijacking、醫療幻覺等） |
| `redteam/redteam_retest.yaml` | 修復後回歸測試 |
| `redteam/provider.py` | Promptfoo 自訂 provider（呼叫本專案 Gemini API） |
| `redteam/run_redteam.sh` | 執行腳本 |
| `redteam/results.json` | 測試結果 |
| `redteam/紅隊攻擊測試報告.md` | 完整報告 |

**執行紅隊測試**：
```bash
cd redteam
bash run_redteam.sh
```

### 幻覺測試（`scripts/test_hallucination.py`）

獨立的 AI 幻覺測試腳本，測試兩種模式：
- 一般諮詢模式（GeminiService）
- 症狀識別模式（TriageService）

```bash
python scripts/test_hallucination.py              # 全部測試
python scripts/test_hallucination.py --mode triage # 只測分流
python scripts/test_hallucination.py --rounds 3    # 每題跑 3 次
```

### Security Prompt A/B 測試（`scripts/ab_test_security_prompt.py`）

比較有/無 security prompt 對正常回覆的影響（時間、品質）。

---

## 工具腳本

| 腳本 | 用途 |
|------|------|
| `scripts/test_hallucination.py` | AI 幻覺測試 |
| `scripts/ab_test_security_prompt.py` | 安全 prompt A/B 測試 |
| `scripts/test_doctor_recommend.py` | 醫師推薦功能測試 |
| `scripts/test_notify.py` | 異常通知推送測試 |
| `scripts/preview_notify_messages.py` | 預覽通知訊息內容 |
| `scripts/export_bound_users.py` | 匯出已綁定用戶 |
| `scripts/test_send_reminder.py` | 就診提醒測試 |
| `scripts/his_mock_server.py` | HIS 系統 mock server |
| `scripts/migrate_json_to_pg.py` | JSON → PostgreSQL 資料遷移 |
| `scripts/check_token_usage.sh` | Token 使用量查詢 |
| `scripts/count_bound_users.sh` | 統計已綁定用戶數 |
| `tools/create_admin.py` | 建立管理後台帳號 |
| `tools/find_place_ids.py` | Google Place ID 查詢 |
| `tools/google_review_response_time.py` | Google 評論回覆時間分析 |

---

## 前端

| 路徑 | 用途 |
|------|------|
| `frontend/liff/bind/index.html` | LIFF 用戶綁定頁面（身分證 + 手機驗證） |

---

## 常見陷阱

- **不要**在 Gunicorn 多 worker 環境下使用 in-memory session，必須用 Redis
- **不要**在 `post_fork` 以外的地方啟動 APScheduler，否則每個 worker 都會跑排程
- **LINE webhook 驗證**：signature 驗證失敗會 abort(400)，確保 Channel Secret 正確
- **Gemini Search Grounding**：每日免費額度 1500 次，`_should_use_search()` 控制是否啟用
- **PostgreSQL 連線池**：`minconn=2, maxconn=10`，不要在 request 外持有連線
- **安全 prompt**：`security_prompt.txt` 會自動合併到所有 bot，修改時注意影響範圍
- **紅隊測試**：修改 security prompt 或 Gemini 服務邏輯後，務必跑 `redteam/run_redteam.sh` 回歸測試
