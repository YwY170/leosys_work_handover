# Chatbot Platform — 完整程式架構說明

> 本文件供交接人員或 AI 開發工具（Claude Code、GitHub Copilot 等）作為上下文使用，以便快速理解專案全貌並接續開發。

---

## 1. 專案概述

這是一個 **多租戶 AI 聊天機器人 SaaS 平台**（內部代號 Leo AI Genie），允許不同品牌客戶透過設定檔即可擁有獨立的 AI 聊天機器人，無需撰寫程式碼。

核心特色：
- **Config-Driven**：所有租戶行為（服務、外觀、提示詞）皆由 JSON 設定驅動
- **可嵌入式**：一行 `<script>` 即可將聊天機器人嵌入任何網站
- **多語言**：自動偵測使用者語言並切換 UI + 回應語言（支援 7 種語言）
- **多服務類型**：對話、查詢、路線規劃、ERP 整合
- **管理後台**：視覺化管理品牌、服務、外觀、數據統計

---

## 2. 技術棧

| 層級 | 技術 |
|------|------|
| 前端 | Next.js 16 (App Router) + React 19 + Tailwind CSS 4 |
| 後端 Chat API | Python Flask + Gunicorn (port 5000) |
| 後端 Admin API | Python Flask + Gunicorn (port 5001) |
| AI 引擎 | Google Gemini 2.5 Flash (google-genai SDK) |
| Session 管理 | Redis |
| 資料庫 | PostgreSQL 16 + SQLAlchemy |
| 反向代理 | Nginx |
| 容器化 | Docker Compose (5 services) |
| ERP 整合 | Frappe REST API |

---

## 3. 目錄結構

```
Chatbot Platform/
├── app/                          # Next.js App Router 頁面
│   ├── admin/                    # 管理後台頁面
│   │   ├── layout.tsx            # 管理後台 Layout（側邊欄）
│   │   ├── login/page.tsx        # 登入頁
│   │   ├── page.tsx              # 重定向到 /admin/tenants
│   │   ├── stats/page.tsx        # 品牌數據總覽
│   │   └── tenants/              # 品牌管理
│   │       ├── page.tsx          # 品牌列表 + 嵌入碼
│   │       └── [id]/             # 單一品牌
│   │           ├── page.tsx      # 品牌基本設定 + 翻譯
│   │           ├── appearance/   # 外觀設定（即時預覽）
│   │           ├── dashboard/    # 數據儀表板
│   │           └── services/     # 服務設定 CRUD
│   ├── api/
│   │   ├── chat/route.ts         # (placeholder，未使用)
│   │   └── chat-widget/route.ts  # 嵌入式 JS 產生器
│   ├── chat/page.tsx             # 聊天頁面入口
│   ├── layout.tsx                # Root Layout (LanguageProvider)
│   └── page.tsx                  # 首頁（重定向到 /admin）
├── src/
│   ├── components/chat/          # 聊天 UI 元件
│   │   ├── ChatContainer.tsx     # 主容器（狀態管理、API 呼叫）
│   │   ├── ChatHeader.tsx        # 標題列
│   │   ├── MessageList.tsx       # 訊息列表
│   │   ├── InputBar.tsx          # 輸入框
│   │   └── QuickActions.tsx      # 快速操作按鈕
│   ├── contexts/
│   │   └── LanguageContext.tsx   # 語言狀態 Context
│   ├── lib/
│   │   ├── api-client.ts         # Chat API Client
│   │   ├── admin/api-client.ts   # Admin API Client
│   │   ├── i18n.ts               # 多語言工具
│   │   └── appearance.ts         # 外觀型別定義
│   ├── locales/
│   │   └── translations.json     # 前端 UI 翻譯檔
│   └── types/                    # TypeScript 型別
├── backend/
│   ├── app.py                    # Chat API 主程式 (Flask)
│   ├── admin_app.py              # Admin API 主程式 (Flask)
│   ├── db.py                     # PostgreSQL 資料層
│   ├── config/
│   │   ├── tenant_manager.py     # 租戶設定管理器
│   │   ├── service_factory.py    # 服務工廠
│   │   ├── tenants.json          # 租戶設定檔（核心）
│   │   └── frappe_templates.py   # Frappe 模組模板
│   ├── middleware/
│   │   └── tenant_auth.py        # 租戶驗證裝飾器
│   ├── services/
│   │   ├── base_gemini_service.py  # Gemini 基礎服務（核心）
│   │   ├── chat_service.py         # 通用對話服務
│   │   ├── query_service.py        # 資料查詢服務
│   │   ├── smart_route_service.py  # 路線規劃服務
│   │   └── frappe_query_service.py # ERP 查詢服務
│   ├── prompts/                  # 各租戶提示詞 (Markdown)
│   │   ├── demo/
│   │   ├── caesarpark/
│   │   ├── tourism/
│   │   └── ...
│   ├── translations/             # AI 生成的翻譯檔
│   ├── public/images/tenants/    # 租戶上傳圖片
│   ├── Dockerfile                # Chat API Docker
│   ├── Dockerfile.admin          # Admin API Docker
│   └── requirements.txt          # Python 依賴
├── docs/
│   ├── ARCHITECTURE.md           # 本文件
│   └── DEPLOYMENT.md             # 部署說明
├── docker-compose.yml            # 容器編排
├── nginx.conf                    # Nginx 反向代理設定
├── Dockerfile                    # 前端 Docker
├── CLAUDE.md                     # Claude Code 入口
├── .github/copilot-instructions.md  # Copilot 入口
├── .env                          # 根目錄環境變數
├── .env.example                  # 環境變數範例
└── package.json                  # 前端依賴
```

---

## 4. 核心架構設計

### 4.1 多租戶架構

```
使用者請求 → Nginx → 前端/後端
                         ↓
              X-Tenant-ID Header 識別租戶
                         ↓
              tenant_auth.py 驗證 + 注入 tenant config
                         ↓
              service_factory.py 動態建立服務實例
                         ↓
              對應 Service 處理請求
```

**租戶設定結構** (`backend/config/tenants.json`)：
```json
{
  "tenant_id": {
    "name": "品牌名稱",
    "api_key_env": "TENANT_XXX_GEMINI_API_KEY",
    "enabled": true,
    "services": { ... },
    "quick_actions": [ ... ],
    "appearance": { ... }
  }
}
```

### 4.2 服務層（Strategy Pattern）

```
BaseGeminiService (核心)
├── ChatService          → 通用對話
├── QueryService         → 資料查詢（含快取）
├── SmartRouteService    → 路線規劃（Maps/Search 自動切換）
└── FrappeQueryService   → ERP 整合（繼承 ChatService）
```

**BaseGeminiService 核心功能**：
- Gemini 2.5 Flash API 呼叫
- Redis Session 管理（2 輪對話歷史，1 小時 TTL）
- Google Search Grounding（網路搜尋增強）
- URL Context（指定網域內容抓取）
- Google Maps Grounding（地圖/導航）
- 多語言回應（自動注入語言指令）
- 並行 URL 處理（ThreadPoolExecutor）
- url_context 失敗自動 fallback 到 google_search

### 4.3 前端聊天流程

```
使用者輸入訊息
    ↓
1. 語言偵測 (POST /api/detect-language)
    ↓
2. 如果非中文 → 載入翻譯版 tenant config
    ↓
3. AI 意圖判斷 (POST /api/chat/intent)
    ↓
4. 根據意圖選擇服務 → 發送聊天請求 (POST /api/chat)
    ↓
5. 顯示回應 + 參考連結
```

### 4.4 嵌入式 Widget

`/api/chat-widget?tenant_id=xxx` 回傳一段 JavaScript：
1. 載入租戶設定取得 chatIconUrl
2. 建立浮動按鈕（右下角圓形圖示）
3. 點擊後開啟 iframe 載入 `/chat?tenant_id=xxx`
4. 支援展開/收合、關閉
5. 支援 GTM 嵌入方式

### 4.5 資料庫設計

```sql
-- 訊息記錄
message_logs (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(64) NOT NULL,
    session_id VARCHAR(64) NOT NULL,
    direction VARCHAR(4) NOT NULL,       -- 'bot' or 'user'
    service_name VARCHAR(64),
    response_ms INTEGER DEFAULT 0,
    lang VARCHAR(16),
    created_at TIMESTAMP DEFAULT NOW()
)

-- 對話 Session 彙總
chat_sessions (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(64) NOT NULL,
    session_id VARCHAR(64) NOT NULL,
    lang VARCHAR(16),
    first_seen TIMESTAMP DEFAULT NOW(),
    last_seen TIMESTAMP DEFAULT NOW(),
    message_count INTEGER DEFAULT 0,
    UNIQUE(tenant_id, session_id)
)
```

---

## 5. API 端點一覽

### Chat API (port 5000)

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/health` | 健康檢查 |
| POST | `/api/detect-language` | 語言偵測 |
| POST | `/api/chat/intent` | AI 意圖判斷 |
| POST | `/api/chat` | 統一聊天接口 |
| GET | `/api/tenant/config` | 租戶前端設定（支援 ?lang= 參數） |
| GET | `/api/query/data` | QueryService 資料查詢 |

### Admin API (port 5001)

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/admin/health` | 健康檢查 |
| GET/POST | `/api/admin/tenants` | 租戶列表/建立 |
| GET/PUT/DELETE | `/api/admin/tenants/<id>` | 租戶 CRUD |
| GET/POST | `/api/admin/tenants/<id>/services` | 服務列表/新增 |
| GET/PUT/DELETE | `/api/admin/tenants/<id>/services/<name>` | 服務 CRUD |
| GET/PUT | `/api/admin/tenants/<id>/quick-actions` | 快速操作管理 |
| GET/PUT | `/api/admin/tenants/<id>/appearance` | 外觀設定 |
| POST | `/api/admin/tenants/<id>/appearance/upload-icon` | 圖示上傳 |
| GET/POST | `/api/admin/tenants/<id>/translations` | 翻譯狀態/生成 |
| GET | `/api/admin/stats/dashboard` | 全域統計 |
| GET | `/api/admin/stats/<id>/daily` | 租戶每日統計（支援 ?days= 參數） |
| GET | `/api/admin/service-classes` | 可用服務類別 |
| GET | `/api/admin/frappe/templates` | Frappe 模組模板 |

---

## 6. 服務設定結構

每個服務在 `tenants.json` 中的結構：

```json
{
  "service_id": {
    "name": "服務顯示名稱",
    "icon": "💬",
    "enabled": true,
    "class": "ChatService | QueryService | SmartRouteService | FrappeQueryService",
    "prompt_file": "prompts/tenant_id/service.md",
    "temperature": 0.7,
    "use_grounding": true,
    "search_keyword": "搜尋關鍵字（Grounding 時附加）",
    "allowed_domains": "domain1.com,domain2.com",
    "loading_message": "處理中（聊天等待訊息）",
    "query_loading_message": "查詢資料中（QueryService 專用）",
    "mode_message": "切換模式時的提示訊息",
    "show_mode_message": false,
    "frappe": {
      "url": "https://erp.example.com",
      "api_key": "...",
      "api_secret": "...",
      "queries": [
        {
          "doctype": "Item",
          "fields": ["item_code", "item_name"],
          "search_fields": ["item_name"],
          "static_filters": [["item_group", "=", "Drug"]],
          "limit": 20
        },
        {
          "doctype": "Bin",
          "link_field": "item_code",
          "parent_field": "item_code",
          "merge_key": "item_code",
          "alias": "stock",
          "fields": ["actual_qty", "warehouse"]
        }
      ]
    }
  }
}
```

---

## 7. 關鍵設計決策

1. **API Key 不存 JSON**：租戶的 Gemini API Key 存在 `.env` 檔案中，`tenants.json` 只存環境變數名稱 (`api_key_env`)，避免敏感資訊外洩。

2. **Hot Reload**：`TenantManager.get_tenant()` 每次呼叫都重新讀取 JSON（開發模式），生產環境可考慮加快取。

3. **非同步 Log**：使用 `ThreadPoolExecutor(max_workers=4)` 非同步寫入 DB，不阻塞 API 回應。

4. **Session 管理**：Redis key 格式 `gemini_session:{tenant_id}:{service_name}:{user_id}`，保留最近 2 輪對話，TTL 1 小時。

5. **Fallback 機制**：`url_context` 抓取失敗時自動 fallback 到純 `google_search`。

6. **意圖分類**：前端先呼叫 `/api/chat/intent` 判斷使用者意圖，再帶 `mode` 參數呼叫 `/api/chat`。

7. **翻譯機制**：Admin 後台可一鍵生成各語言翻譯檔（用 Gemini 翻譯），前端根據偵測到的語言載入對應翻譯覆蓋 config。翻譯檔帶 `_source_hash` 追蹤原始文字是否異動。

8. **Docker 中 tenants.json 路徑**：生產環境使用 `backend/data/tenants.json`（volume 掛載），而非 `backend/config/tenants.json`。

---

## 8. 新增租戶/服務的流程

### 新增租戶
1. Admin 後台 → 品牌管理 → 新增品牌
2. 系統自動：
   - 建立 `tenants.json` 條目
   - 寫入 `.env` API Key（格式：`TENANT_{ID大寫}_GEMINI_API_KEY`）
   - 建立預設 `general` 服務
   - 建立 `prompts/{tenant_id}/general.md` 提示詞檔案

### 新增服務
1. Admin 後台 → 品牌 → 服務設定 → 新增服務
2. 選擇服務類別（ChatService / QueryService / SmartRouteService / FrappeQueryService）
3. 設定參數（temperature、grounding、search_keyword、allowed_domains 等）
4. 撰寫提示詞（Markdown 格式）
5. 系統自動加入 Quick Actions（general 除外）

---

## 9. 開發注意事項

- 前端環境變數為 **build-time**（`NEXT_PUBLIC_*`），修改後需重新 build
- 後端兩個 Flask app 共用 `config/`、`services/`、`middleware/` 模組
- `tenants.json` 是核心設定檔，所有 CRUD 操作最終都寫入此檔案
- Prompt 檔案放在 `backend/prompts/{tenant_id}/{service_id}.md`
- 翻譯檔放在 `backend/translations/{tenant_id}/{lang}.json`
- 圖片上傳到 `backend/public/images/tenants/{tenant_id}/`（Docker 中掛載到 nginx）
- 程式碼風格：Python 用中文註解，TypeScript 用繁體中文 UI 文字
- 所有 API 回應格式統一為 JSON
- 錯誤處理：後端回傳 `{"error": "訊息"}` + 對應 HTTP status code
- 管理後台登入預設 key: `admin_secret_key`（生產環境務必更改）
