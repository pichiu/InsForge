# 偵察報告 (Reconnaissance)

> Trace 產出，來源 commit: `main`@`0dd55c5` (見 `.trace/TRACE_META.md`)

## 1. 專案一句話總結

InsForge 是一個開源、all-in-one 的 **backend-as-a-service 平台**，設計目標是給「coding agent」（而非人類開發者）直接操作，讓 agent 透過 MCP Server 或 CLI 取得資料庫、認證、儲存、compute、hosting、AI gateway 等後端能力，端到端交付全端應用。

## 2. 套件管理與 Monorepo 結構

- 根目錄 `package.json`：npm workspaces = `backend`, `frontend`, `packages/*`
- Build 系統：**Turborepo** (`turbo.json`)，任務：`build` / `typecheck` / `test` / `lint` / `dev` / `clean`
- Node.js ESM (`"type": "module"`)，TypeScript 5.8
- `packageManager: npm@11.3.0`

Workspace 明細：

| workspace | package name | 定位 |
|---|---|---|
| `backend/` | `insforge-backend` | Express.js API server，核心後端邏輯 |
| `frontend/` | `insforge-dashboard`（app 名稱與 package 目錄同名但非同一套件） | self-hosting 用的本地 shell，掛載 `@insforge/dashboard` |
| `packages/dashboard/` | `@insforge/dashboard` | 可發佈的共用 dashboard package，支援 self-hosting 與 cloud-hosting 兩種模式 |
| `packages/ui/` | `@insforge/ui` | React 元件庫、design tokens、Tailwind preset |
| `packages/shared-schemas/` | `@insforge/shared-schemas` | Zod schema + TS 型別，前後端共用契約 |
| `functions/` | (無 package.json，Deno runtime) | 使用者自訂 Deno edge functions 執行環境 + 範例 |

> ⚠️ 容易搞混：`frontend/` 是「自架版殼層」，`packages/dashboard/` 才是「真正的 dashboard 功能實作」。這是 insforge-dev skill 明確強調的 package boundary。

## 3. 技術棧總覽

| 類別 | 技術 | 版本（依 package.json） | 用途 |
|---|---|---|---|
| Runtime | Node.js | ESM, TS 5.8 | 後端執行環境 |
| Web Framework | Express.js | ^4.22.0 | HTTP API server |
| 資料庫 | PostgreSQL | `ghcr.io/insforge/postgres:v15.13.4`（含 pgvector） | 主要關聯式資料庫 |
| REST 層 | PostgREST | `postgrest/postgrest:v12.2.12` | 自動由 Postgres schema 產生 REST API（給 SDK/agent 用） |
| Migration | node-pg-migrate | ^8.0.3 | SQL migration 版本控管，schema=`system` |
| Auth | 自研 JWT + jose + jsonwebtoken + bcryptjs | — | Email/password、OAuth（Google/GitHub/Discord/Microsoft/LinkedIn/X/Apple） |
| 即時通訊 | socket.io | ^4.8.1 | Realtime 訂閱/推播 |
| 檔案儲存 | AWS SDK v3（S3, CloudFront signer） | — | S3 相容物件儲存（含 local disk fallback） |
| Edge Functions | Deno | `denoland/deno:alpine-2.0.6` | 使用者自訂 serverless function runtime，獨立容器 |
| AI Gateway | openai SDK + 自研 provider 層 | ^5.19.1 | OpenAI 相容 API，代理多家 LLM provider（透過 OpenRouter 等） |
| 金流 | stripe, razorpay | — | Payments provider（雙供應商） |
| Email | nodemailer | — | Email 發送 |
| 前端框架 | React 19 + Vite 7 | — | Dashboard UI |
| CSS | Tailwind CSS v4 | — | 樣式系統，含 `@insforge/ui` 的 tailwind preset |
| Schema 驗證 | Zod | ^3.23.8 | 前後端共用型別驗證（`shared-schemas`） |
| 測試 | Vitest（單元/整合）、supertest、Playwright（UI smoke，dashboard） | — | |
| CI/CD | GitHub Actions（`.github/workflows/`） | — | |
| 容器化 | Docker + docker-compose（多個 compose 檔：`dev`/`prod`/`dokploy`） | — | |
| 部署 | Railway / Zeabur / Sealos / Vercel（前端）/ Fly.io（compute） | — | 一鍵部署選項 |
| OpenAPI | `zod-to-openapi` | — | 自動由 Zod schema 產生 OpenAPI 文件（`openapi/` 目錄） |

## 4. 目錄結構快照（3 層）

```
InsForge/
├── backend/                     # Express API server（monolith，內部依 service 切分）
│   ├── src/
│   │   ├── api/
│   │   │   ├── middlewares/     # error handling、rate limiter、auth guard 等
│   │   │   └── routes/          # 22 個功能路由群組（見下）
│   │   ├── infra/                # config / database / realtime / security / socket
│   │   ├── providers/            # 外部整合的底層 client 封裝
│   │   ├── services/             # 業務邏輯層（對應每個 route 群組）
│   │   ├── types/
│   │   └── utils/
│   ├── scripts/
│   └── tests/                    # cloud / integration / local / manual / unit
├── frontend/                     # self-hosting 殼層（掛載 packages/dashboard）
│   ├── src/
│   │   ├── cloud-hosting/
│   │   └── self-hosting/
│   └── public/
├── packages/
│   ├── dashboard/                # 可發佈 dashboard 套件（真正的 UI 功能實作）
│   │   ├── src/
│   │   └── tests/
│   ├── ui/                       # 設計系統元件庫
│   │   └── src/
│   └── shared-schemas/           # 前後端共用 Zod schema/型別
│       └── src/
├── functions/                    # Deno edge functions（使用者自訂）+ examples
├── docs/                         # 產品文件（Mintlify，docs.json 設定）
│   ├── core-concepts/            # ai / analytics / authentication / compute / database /
│   │                             #   functions / messaging / payments / realtime / sites / storage
│   ├── sdks/ (typescript / kotlin / swift / rest)
│   ├── agent-native/, examples/, showcase/, superpowers/
├── openapi/                      # OpenAPI 產出
├── deploy/                       # docker-compose 變體、docker-init（DB bootstrap SQL）、zeabur 設定
├── .agents/docs/                 # agent 用的補充說明文件（deployment, payments, realtime 等）
├── .internal/docs/               # 內部維運文件：audits / plans / specs（非公開產品文件）
├── .claude/, .codex/, .agents/   # AI coding agent 的 skills（insforge-dev, doc-author）
├── examples/                     # 範例應用（oauth、python-ml-experiment-tracker）
└── scripts/                      # repo 層級腳本
```

### 架構模式判斷

**Monorepo + Monolithic backend + 可發佈 UI package + 多容器部署（poly-service via docker-compose）**：
- Backend 本身是單一 Express monolith，但內部依 `services/` + `providers/` 做分層（service 呼叫 provider，provider 封裝外部 SDK）— 類似 **hexagonal / ports-and-adapters** 的精神，但非嚴格框架化的 hexagonal。
- 整體部署拓樸是 **多容器編排**（postgres + postgrest + insforge(app) + deno），backend app 與 PostgREST 併存、各自提供不同用途的 API 層（app 端做「智慧語意層」，PostgREST 做「原始 REST passthrough」）。
- Frontend 採 **shared package + host shell** 模式：`packages/dashboard` 是核心，`frontend/` 只是 self-hosting 用的最小 host，cloud 版本應該有另一個 host（未在本 repo 內，屬 closed-source cloud 部分）。
- Edge Functions 走 **獨立 runtime 容器**（Deno），透過 HTTP 從 backend app 觸發，是外部整合而非同進程執行。

## 5. 既有文件掃描摘要

### 5.1 `README.md`（根目錄）
權威、對外的專案入口文件。內容：專案定位、MCP/CLI 兩種 agent 介面、core products（Auth/DB/Storage/Model Gateway/Edge Functions/Compute[private preview]/Site Deployment）、quickstart（cloud 或 docker-compose self-host）、一鍵部署選項（Railway/Zeabur/Sealos）。

### 5.2 `CONTRIBUTING.md`
- Project structure 段落與實際目錄大致相符，但描述較簡略（未提及 `packages/ui`、`packages/dashboard` 的 self-hosting/cloud-hosting 雙模式細節，只籠統寫 `/frontend`）。
- Issue-first workflow：需先 claim issue（有維護者 agent「章北海 Zhang Beihai」自動指派），每人最多同時 3 個 open assigned issues（跨全部 InsForge repo）。
- Branch 命名慣例：`feat/`, `fix/`, `docs/`, `refactor/`, `test/`, `chore/`。
- PR 前必須 `npm run test:e2e` + `npm run lint`。

### 5.3 `docs/`（Mintlify 產品文件，`docs.json` 設定）
公開產品文件，涵蓋 core-concepts（ai/analytics/authentication/compute/database/functions/messaging/payments/realtime/sites/storage）、多語言 SDK（typescript/kotlin/swift/rest）、agent-native 使用說明、examples、showcase。這是最權威的「功能規格」來源，但偏產品/使用者導向，非程式碼實作導向。

### 5.4 `.agents/docs/`（agent 專用補充文件）
`deployment.md`, `insforge-instructions-sdk.md`, `payments-razorpay.md`, `payments-stripe.md`, `payments.md`, `real-time.md` — 給 AI coding agent 在使用 InsForge 當後端時參考的操作說明，非本專案原始碼架構文件。

### 5.5 `.internal/docs/`（內部文件，非公開）
`audits/`, `plans/`, `specs/` — 內部維運/規劃文件。⚠️ 未逐一讀取內容（屬內部/可能含決策草案而非最終狀態），本次 trace 僅記錄其存在，不引用其內容作為架構依據。

### 5.6 `.claude/skills/insforge-dev/`（本 repo 的 AI agent skill，最具體的「維護者心智模型」）
明確定義 package boundary 與開發慣例（详见 `.trace/_context/recon.md` 之前已讀取內容，摘要如下）：
- Contract 變更先動 `packages/shared-schemas/`，再動 consumer。
- Backend 分層：route → service → provider/infra。
- Dashboard 共用行為都放 `packages/dashboard/`；`frontend/` 只放 self-hosting-only 的 bootstrap/env/樣式。
- Backend TS 用 ESM 風格 `.js` import specifier（即使原始檔是 `.ts`）。
- Backend 成功回應通常是 raw JSON，不包 `{ data }`。
- 驗證用共用 Zod schema + `AppError`。
- Dashboard 資料存取一律經 `apiClient` + React Query。
- 禁止使用 TypeScript `any`。
- PR 前檢查清單：`npx turbo run typecheck` → `npx turbo run lint` → `npx turbo run test` → （若跨套件）`npx turbo run build` → e2e gate。

**這份 skill 文件的可信度視為最高**（是維護者自己維護、給 agent 用的即時規範），本次 trace 的其餘文件會與程式碼交叉驗證，若發現落差記錄於 `DISCOVERY_LOG.md`。

## 6. 落差初步觀察（詳細清單見 Stage 3 的 DISCOVERY_LOG.md）

- `CONTRIBUTING.md` 的 Project Structure 段落未提及 `packages/ui`、`packages/dashboard`、`functions/` 的角色，對新貢獻者來說可能造成「不知道 UI 元件庫在哪」的困惑。
- README 提及 Compute 為 "private preview"，但程式碼中 `backend/src/services/compute/`、`backend/src/providers/compute/`、`backend/src/api/routes/compute/` 已有完整實作（見 docker-compose 中 `FLY_API_TOKEN`/`FLY_ORG` 環境變數），需在 Stage 2 進一步確認其成熟度與是否有 feature flag 保護。

## 7. Entry Point 初步線索（供 Stage 2 使用）

- `backend/src/server.ts`：Express app 組裝點，import 了 22 個 route 群組（auth/database/storage/metadata/logs/docs/functions/secrets/usage/ai/memory/realtime/email/deployments/webhooks/s3-gateway/payments/advisor + 未列出的 analytics/compute/schedules/webscraper）。
- `backend/src/infra/database/migrations/`：SQL migration 從 `000_create-base-tables.sql` 開始，schema 名稱 `system`（透過 `node-pg-migrate`）。
- Docker Compose 啟動指令：`npm install && turbo build（shared-schemas/ui/dashboard）→ migrate:up → concurrently 跑 backend + frontend dev server`。
