# InsForge — 專案總覽與速查

> 本文件由自動化 trace 產生，來源 commit：`main`@`0dd55c5`（見 [`TRACE_META.md`](./TRACE_META.md)）。

## 這是什麼

InsForge 是一個開源、all-in-one 的 **backend-as-a-service 平台**，設計目標是給「AI coding agent」（而不是人類開發者）直接操作：agent 透過 **MCP Server**（自架版與雲端版皆有）或 **CLI + Skills**（僅雲端版）取得資料庫（Postgres）、認證、儲存（S3 相容）、compute（Fly.io）、edge functions（Deno）、AI model gateway（OpenAI 相容）、部署（Vercel）等後端能力，讓 agent 像後端工程師一樣讀取狀態、下 migration、部署函式、除錯，端到端交付全端應用。授權為 Apache 2.0，原始碼公開於 GitHub。

## 技術棧總覽

| 類別 | 技術 | 版本 | 用途 |
|---|---|---|---|
| Monorepo / Build | npm workspaces + Turborepo | turbo ^2.9 | `backend` / `frontend` / `packages/*` 統一 build/lint/test/typecheck |
| Runtime | Node.js（ESM） | TypeScript ^5.8 | 後端執行環境 |
| Web Framework | Express.js | ^4.22 | HTTP API server（monolith，內部依 route/service/provider 分層） |
| 資料庫 | PostgreSQL（含 pgvector） | `ghcr.io/insforge/postgres:v15.13.4` | 主要關聯式資料庫，agent 可動態建表 |
| REST 直通層 | PostgREST | `v12.2.12` | 由 Postgres schema 自動產生 REST API，與 Express app 並存的第二條資料路徑 |
| Migration | node-pg-migrate | ^8.0.3 | SQL migration 版本控管（`system.migrations`） |
| Auth | 自研 JWT（jose）+ OAuth（7+ providers） | — | Email/Password、Google/GitHub/Discord/Microsoft/LinkedIn/X/Apple/自訂 OIDC |
| Realtime | pg_notify + Socket.IO | ^4.8 | DB 變更透過 LISTEN/NOTIFY 廣播到前端 |
| 檔案儲存 | AWS S3 SDK v3 + CloudFront | — | S3 相容物件儲存，未設定時 fallback 本地磁碟 |
| Edge Functions | Deno（獨立容器） | `alpine-2.0.6` | 使用者自訂 serverless function，per-request 全新 Worker 沙箱 |
| AI Gateway | openai SDK + OpenRouter | ^5.19 | OpenAI 相容 API，代理多家 LLM provider |
| 金流 | Stripe + Razorpay | — | 雙 provider 並列（無共用抽象層） |
| 前端框架 | React 19 + Vite 7 | — | Dashboard UI |
| CSS | Tailwind CSS v4 | — | 樣式系統，`@insforge/ui` 提供共用 preset |
| Schema 驗證 | Zod（`@insforge/shared-schemas`） | ^3.23 | 前後端共用型別與驗證契約 |
| 測試 | Vitest + supertest（單元/整合）、Playwright（dashboard UI smoke） | — | |
| CI/CD | GitHub Actions | — | `.github/workflows/` |
| 容器化/部署 | Docker Compose（dev/prod/dokploy）、Railway/Zeabur/Sealos/Vercel/Fly.io | — | 多種一鍵部署選項 |

## 關鍵指令速查

```bash
# 安裝與啟動（本地開發，Docker Compose，推薦）
cp .env.example .env
docker compose -f docker-compose.prod.yml up   # 或 docker compose up 走 dev target

# Node 原生開發（免 Docker，需自行起 Postgres/PostgREST/Deno）
npm run install:all      # 根目錄 + backend + frontend 各自 npm install
npm run dev              # turbo run dev（並行跑所有 workspace 的 dev script）
npm run dev:backend      # 只跑 backend（tsx watch）
npm run dev:frontend     # 只跑 frontend（vite）

# 建置 / 檢查
npm run build            # turbo run build
npm run typecheck        # turbo run typecheck
npm run lint              # turbo run lint
npm run lint:fix          # eslint . --fix
npm run format            # prettier --write .

# 測試
npm run test              # turbo run test
npm run test:backend      # cd backend && npm test（vitest）
npm run test:e2e          # cd backend && ./tests/run-all-tests.sh

# 資料庫 migration（backend/ 目錄下）
npm run migrate:up        # bootstrap + node-pg-migrate up（schema=system）
npm run migrate:down
npm run migrate:create
```

> Pre-PR 完整檢查清單（維護者慣例，見 `.claude/skills/insforge-dev/`）：
> `npx turbo run typecheck` → `npx turbo run lint` → `npx turbo run test` →（跨套件變更時）`npx turbo run build` → e2e gate。

## 文件地圖

| 文件 | 內容 |
|---|---|
| [`INDEX.md`](./INDEX.md) | 本文件：專案總覽、技術棧、指令速查 |
| [`CODEBASE_MAP.md`](./CODEBASE_MAP.md) | 目錄地圖、「我想改 X 要看哪裡」速查表、模組依賴圖 |
| [`ARCHITECTURE.md`](./ARCHITECTURE.md) | 系統架構、元件清單、通訊模式、關鍵設計決策 |
| [`DATA_MODEL.md`](./DATA_MODEL.md) | 核心 entity、ER diagram、schema 摘要、migration 機制 |
| [`API_SURFACE.md`](./API_SURFACE.md) | REST API 清單、認證模型、錯誤格式 |
| [`DEV_GUIDE.md`](./DEV_GUIDE.md) | 環境建置、開發 workflow、測試、除錯、貢獻流程 |
| [`DISCOVERY_LOG.md`](./DISCOVERY_LOG.md) | 既有文件與程式碼的落差、TODO/FIXME 彙整、待解問題 |
| [`TRACE_META.md`](./TRACE_META.md) | Trace metadata，供增量更新使用 |
| `_context/*.md` | Stage 1-2 的中間分析檔案（recon / entry_points / data_flow / core_logic / extensions / integrations / configuration / web_findings） |

## 專案專屬術語表

| 術語 | 說明 |
|---|---|
| **Self-hosting mode** | 使用者自行用 Docker Compose 部署 InsForge，`frontend/` 作為最小 host shell 掛載 `packages/dashboard` |
| **Cloud-hosting mode** | InsForge 官方雲端服務（insforge.dev），dashboard 以另一個（非本 repo 內、closed-source）host 掛載，並多了 partner/iframe 通訊層 |
| **Model Gateway** | OpenAI 相容的 AI API，內部代理到 OpenRouter 等多個 LLM provider |
| **Subhosting** | Deno Deploy 的多租戶部署 API，InsForge 用它來部署使用者的 edge functions 到雲端；本地開發則走 `functions/server.ts` 起的本地 Deno runtime |
| **PostgREST 直通層** | 與 Express app 並存的第二條資料路徑，直接由 Postgres schema 生成 REST API，供使用者的應用程式（非 dashboard）以 RLS 保護的方式存取自己的資料表 |
| **`system` schema** | InsForge 內部管理用的 Postgres schema（migrations、secrets 等），與使用者自建的 `public` schema 資料表分開 |
| **Worker template（沙箱）** | `functions/worker-template.js`，每次 edge function 呼叫都建立一個全新、一次性的 Deno Worker，執行前會鎖死 `Deno.env`/`process.env` 防止逃逸 |
| **`getInstance()` 單例模式** | Backend 全面採用的依賴組裝方式（非 DI container），152+ 個檔案使用此模式 |
| **AppError** | Backend 統一的錯誤型別，由 `errorMiddleware` 轉換成一致的 JSON 錯誤格式 |
| **`#imports` map** | `packages/dashboard/package.json` 定義的 Node subpath imports（`#features/*`、`#lib/*` 等），是 dashboard 內部模組邊界的慣例 |
| **章北海（Zhang Beihai）** | 專案維護者用的 GitHub repo-maintainer agent，負責 issue 自動指派 |

## 延伸閱讀

- 官方文件：[docs.insforge.dev](https://docs.insforge.dev/introduction)
- 官方部落格（2.0 launch）：[insforge.dev/blog/insforge-launch-v2](https://insforge.dev/blog/insforge-launch-v2)
- 完整搜尋結果與來源見 [`_context/web_findings.md`](./_context/web_findings.md)
