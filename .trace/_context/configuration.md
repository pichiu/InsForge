# Configuration 與 Environment Loading 追蹤筆記

> 這是一份 intermediate working doc，供追蹤 InsForge monorepo 的 config/env 載入機制使用。粗略對照 file:line，未經進一步 polish。

## 1. Config 載入優先順序 — `backend/src/infra/config/app.config.ts`

檔案結構：
- `app.config.ts:1-21` 先做自己的 dotenv 載入（見第 2 節），接著才 `export interface AppConfig {...}`（`app.config.ts:23-106`）定義完整型別。
- `loadConfig()`（`app.config.ts:134-237`）是唯一的組裝函式，pattern 一律是 `process.env.XXX || <default>`，沒有用任何 schema validation library（例如 zod）去驗證這層，型別靠 TS interface 而非 runtime validation。
- 模組載入時就立即執行 `export const appConfig: AppConfig = loadConfig();`（`app.config.ts:239`），也就是 **module import 時就會 snapshot 一次環境變數**，之後即使 `process.env` 再變動，`appConfig` 也不會重新讀取（除非重啟 process）。

輔助 parser：
- `parseEnvInt`（`app.config.ts:108-115`）：非數字或 <=0 或非安全整數則 fallback。
- `parseEnvBool`（`app.config.ts:117-120`）：只認 `'1'|'true'|'yes'|'on'`（不分大小寫），其餘一律 false。
- `parseEnvBytes`（`app.config.ts:124-132`）：驗證純數字字串，並用 `AWS_MAX_SINGLE_PUT_BYTES = 5*1024^3` 做上限 clamp（`app.config.ts:122,131`）。

`server.ts` 對 `appConfig` 的具體使用（節錄，見 `backend/src/server.ts`）：
- `server.ts:79` `app.set('trust proxy', appConfig.server.trustProxy)` — 由 `parseTrustProxySetting(process.env.TRUST_PROXY)` 決定（`app.config.ts:181`，實作在 `backend/src/utils/trust-proxy.ts`，⚠️ 未細看該檔案邏輯）。
- `server.ts:172-173` `appConfig.server.maxJsonBodySize` / `maxUrlencodedBodySize` 用來設定 express body-parser 上限（對應 `MAX_JSON_BODY_SIZE` / `MAX_URLENCODED_BODY_SIZE`，預設 `100mb` / `10mb`）。
- `server.ts:236` `appConfig.functions.denoRuntimeUrl` 當作本地 Deno runtime fallback URL（`DENO_RUNTIME_URL`，預設 `http://localhost:7133`；docker-compose 覆寫成 `http://deno:7133`，見第 7 節）。
- `server.ts:325` `const PORT = appConfig.app.port;`（`PORT` 環境變數，預設 `7130`）。

## 2. dotenv 載入機制（重點：載入了兩次，各自獨立）

**`app.config.ts:1-21`**（第一次載入，且是 module 被 import 時最早執行的程式碼之一）：
```
const envPaths = [
  path.resolve(__dirname, '../../../../.env'),   // 依 dist/ 結構回推
  path.resolve(__dirname, '../../../.env'),
  path.resolve(process.cwd(), '.env'),
  path.resolve(process.cwd(), '../.env'),
];
const envPath = envPaths.find((p) => fs.existsSync(p));
if (envPath) { dotenv.config({ path: envPath }); } else { dotenv.config(); }
```
用 4 個候選路徑輪詢找 repo root 的 `.env`（考慮 build 後 `dist/` 目錄層數不同），找不到則 fallback 用 dotenv 預設行為（往 cwd 找）。

**`server.ts:51-58`**（第二次、獨立的載入）：
```
const envPath = path.resolve(__dirname, '../../.env');   // 只嘗試一個路徑
if (fs.existsSync(envPath)) { dotenv.config({ path: envPath }); }
else { dotenv.config(); }
```
只嘗試單一路徑（`backend/../../.env`，即 repo root），找不到才 fallback。

⚠️ 這兩處各自獨立呼叫 `dotenv.config()`，且 `dotenv.config()` 預設不會覆寫已存在的 `process.env` 值（dotenv 的內建行為），所以誰先 import 誰先生效；由於 `server.ts` 會 `import { appConfig } from '@/infra/config/app.config.js'`（`server.ts:45`），依 ESM import 提升規則，**`app.config.ts` 頂層程式碼（含它自己的 dotenv 載入）會先於 `server.ts:51` 執行**，所以實際生效的是 `app.config.ts` 那組更完整的路徑搜尋邏輯；`server.ts` 自己的 dotenv.config() 基本上是重複、對已載入的變數是 no-op。

**`backend/package.json` 的 `dotenv-cli` 用法**：
- `package.json:10` `"dev": "dotenv -e ../.env -- tsx watch src/server.ts"` — 用 `dotenv-cli` 顯式指定 `../.env`（即 repo root）在啟動前注入 env，這是**第三層**載入路徑（CLI 層），會先於上述兩個程式內載入生效。
- `package.json:27` `"migrate:up:local": "dotenv -e ../.env -- npm run migrate:bootstrap && dotenv -e ../.env -- node-pg-migrate up --migrations-dir src/infra/database/migrations --migrations-schema system --migrations-table migrations"`
- `package.json:28` `"migrate:down:local": "dotenv -e ../.env -- node-pg-migrate down ..."`
- devDependency `dotenv-cli@^10.0.0`（`package.json:93`），runtime dependency `dotenv@^16.4.5`（`package.json:56`）。

小結載入優先序（愈前面愈早生效，因 dotenv 預設不覆寫已存在的 process.env）：
1. Docker Compose `environment:` 區塊（container 啟動時就已注入，見第 7 節）／或本機終端機手動 export
2. `dotenv-cli`（`dotenv -e ../.env --`，僅 `npm run dev` / `migrate:*` script 會用到）
3. `app.config.ts:1-21` 內部 dotenv（4 條路徑輪詢）
4. `server.ts:51-58` 內部 dotenv（單一路徑，實際上多半是 no-op）

## 3. `.env.example`（repo root，315 行）完整分區摘要

| 區塊 | 行號 | 關鍵變數 | 必填/選填/功能旗標 |
|---|---|---|---|
| Server Configuration | 13-24 | `PORT`(7130)、`MAX_JSON_BODY_SIZE`(100mb)、`MAX_URLENCODED_BODY_SIZE`(10mb) | 選填，皆有 default |
| PostgreSQL Configuration | 26-33 | `POSTGRES_USER`/`POSTGRES_PASSWORD`/`POSTGRES_DB` | 選填，有 default（postgres/postgres/insforge） |
| Ports (Configurable) | 35-55 | `POSTGRES_PORT`(5432)、`POSTGREST_PORT`(5430)、`APP_PORT`(7130)、`AUTH_PORT`(7131)、`UI_PORT`(7132)、`DENO_PORT`(7133)、`API_BASE_URL`、`VITE_API_BASE_URL` | 選填，皆有 default；`VITE_API_BASE_URL` 是前端 build-time 變數（見第 6 節） |
| Authentication & Security | 57-87 | `JWT_SECRET`（**必填**，需 ≥32 字元）、`ROOT_ADMIN_USERNAME`/`ROOT_ADMIN_PASSWORD`、`ENCRYPTION_KEY`（強烈建議設定，否則 fallback 用 JWT_SECRET，見第 4 節）、`ACCESS_API_KEY`（選填，未提供則自動產生）、`ACCESS_ANON_KEY`（選填，自動產生）、`CLOUD_API_HOST`（選填，只有 cloud 功能才用到） | `JWT_SECRET` 實質必填；`ENCRYPTION_KEY` 強烈建議但技術上選填 |
| Deployment Configuration | 89-96 | `VERCEL_TOKEN`/`VERCEL_TEAM_ID`/`VERCEL_PROJECT_ID` | 功能旗標：self-hosted 網站部署 + 自訂網域才需要；legacy 部署還需搭配 `AWS_S3_BUCKET` |
| AWS Config Bucket | 98-104 | `AWS_CONFIG_BUCKET`(insforge-config)、`AWS_CONFIG_REGION`(us-east-2) | 選填，有 default，用來從 S3 載入遠端設定檔 |
| Storage Configuration | 108-153 | `AWS_S3_BUCKET`（留空=用本地檔案系統儲存，設定=用 S3）、`AWS_REGION`、`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`、`S3_ACCESS_KEY_ID`/`S3_SECRET_ACCESS_KEY`/`S3_ENDPOINT_URL`（S3-compatible 如 MinIO/Wasabi，優先於 AWS_* ）、`S3_FORCE_PATH_STYLE`(true)、`MAX_FILE_SIZE`、`AWS_CLOUDFRONT_URL`/`_KEY_PAIR_ID`/`_PRIVATE_KEY`（選填，CloudFront 簽名 URL，僅原生 AWS S3 適用，設 `S3_ENDPOINT_URL` 時會被忽略） | 功能旗標式：storage 後端依 `AWS_S3_BUCKET` 是否設定切換 local vs S3；CloudFront 三變數需全部到齊才生效，否則 fallback 回 S3 presigned URL |
| Logging Configuration | 155-161 | `LOGS_DIR`（留空 default `./logs`）；若有 `AWS_REGION`+AWS 憑證則走 CloudWatch，否則走 file-based | 功能旗標：依是否有 AWS 憑證切換 CloudWatch vs 檔案 log |
| Anonymous Telemetry | 163-172 | `INSFORGE_TELEMETRY_DISABLED`（設 1 關閉遙測） | 選填 feature flag，預設開啟 |
| AI/LLM Configuration | 174-185 | `OPENROUTER_API_KEY`、`MAX_COMPLETION_TOKENS`(16384) | 選填；沒有 key 則 AI 功能不可用 |
| Stripe Payments | 187-193 | `STRIPE_LIVE_SECRET_KEY`/`STRIPE_TEST_SECRET_KEY` | 選填 feature flag（developer-owned 金鑰） |
| Razorpay Payments | 195-203 | `RAZORPAY_LIVE_KEY_ID`/`_SECRET`、`RAZORPAY_TEST_KEY_ID`/`_SECRET` | 選填 feature flag |
| Analytics | 205-210 | `VITE_PUBLIC_POSTHOG_KEY`（僅本地開發用） | 選填，前端 build-time 變數 |
| OAuth Configuration | 212-261 | 六組 provider：`GOOGLE_CLIENT_ID/SECRET`、`GITHUB_CLIENT_ID/SECRET`、`MICROSOFT_CLIENT_ID/SECRET`、`DISCORD_CLIENT_ID/SECRET`、`LINKEDIN_CLIENT_ID/SECRET`、`X_CLIENT_ID/SECRET`、`APPLE_CLIENT_ID/SECRET`（Apple 的 secret 是 JSON 字串格式） | 每組都是獨立 feature flag：設了該 provider 的 client id/secret 才會啟用該社群登入 |
| Multi-tenant Cloud Configuration | 263-270 | `DEPLOYMENT_ID`、`PROJECT_ID`、`APP_KEY` | 註解明講「only used for the cloud-hosted solution / leave empty for self-hosted」→ cloud-only feature flag（見第 5 節） |
| Serverless Functions (Deno Deploy) | 272-279 | `DENO_DEPLOY_TOKEN`、`DENO_DEPLOY_ORG_ID` | 選填 feature flag，未設則 functions subhosting 不可用（`FunctionService.isSubhostingConfigured()`，見 `secrets/index.routes.ts:23` 的引用） |
| Compute Services (Fly.io) | 281-304 | `FLY_API_TOKEN`+`FLY_ORG`（**兩者必須同時設定**才啟用 self-host Compute）、`COMPUTE_DOMAIN`（選填，自訂 wildcard domain） | Feature flag：self-host 優先於 cloud-mode；cloud-mode compute 則是 `PROJECT_ID`+`CLOUD_API_HOST` 由 insforge-cloud 自動注入（見下方 compute provider 分析） |

Feature-flag 交叉驗證（程式碼證據）：
- `backend/src/services/compute/services.service.ts:250-276`：註解明講「Self-host takes precedence: if FLY_API_TOKEN is set, the user has their own Fly account... (`PROJECT_ID + CLOUD_API_HOST + JWT_SECRET` all present)」，若 `appConfig.fly.apiToken` 有值但 `appConfig.fly.org`（`FLY_ORG`）沒設，會直接報錯要求同時設定兩者（`services.service.ts:255-261`）。
- `secrets/index.routes.ts:22-26` `triggerSecretsRedeployment` 只在 `functionService.isSubhostingConfigured()` 為真時觸發（依賴 `DENO_DEPLOY_TOKEN`/`DENO_DEPLOY_ORG_ID`，⚠️ 未細看 `isSubhostingConfigured()` 內部判斷式）。

## 4. Secrets 加密機制 — `EncryptionManager` + `SecretService`

核心加密工具：`backend/src/infra/security/encryption.manager.ts`
- `getEncryptionKey()`（`encryption.manager.ts:11-27`）：優先讀 `process.env.ENCRYPTION_KEY`，若未設定則 **fallback 用 `process.env.JWT_SECRET`**，並打一條 `logger.warn`（`encryption.manager.ts:17-23`）警告「rotating JWT_SECRET without setting ENCRYPTION_KEY 會讓所有已存 secrets 損毀」。兩者皆未設則直接 throw（`encryption.manager.ts:14-16`）。
- Key 經 `crypto.createHash('sha256').update(key).digest()`（`encryption.manager.ts:24`）轉成 32-byte AES key，並 memoize 在 static 變數 `encryptionKey`（`encryption.manager.ts:9`）— 只算一次，之後 `process.env` 變動不影響（跟 `appConfig` 一樣是 process 生命週期內固定）。
- 演算法：**AES-256-GCM**（`encryption.manager.ts:35,64`）。
  - `encrypt()`（`encryption.manager.ts:32-43`）：隨機 16-byte IV，輸出格式為 `iv(hex):authTag(hex):ciphertext(hex)`，以 `:` 分隔三段。
  - `decrypt()`（`encryption.manager.ts:48-71`）：反解析三段、驗證 authTag 長度必須 16 bytes（`encryption.manager.ts:60-62`），再用 `crypto.createDecipheriv` 解密。

儲存位置與使用者：`backend/src/services/secrets/secret.service.ts`
- 資料表 `system.secrets`，欄位 `value_ciphertext` 存放上述加密字串（例：`createSecret` 於 `secret.service.ts:66-97` 呼叫 `EncryptionManager.encrypt(input.value)`）。
- 用途不只是「使用者自訂的 function secrets」，還包含系統自身敏感值：
  - `API_KEY`（`initializeApiKey`，`secret.service.ts:806-829`，seed 自 `ACCESS_API_KEY` 環境變數或隨機產生 `ik_` 開頭金鑰）
  - `ANON_KEY`（`initializeAnonKey`，`secret.service.ts:768-800`，seed 自 `ACCESS_ANON_KEY` 或隨機產生 `anon_` 開頭金鑰）
  - `JWT_PRIVATE_KEY`/`JWT_PUBLIC_KEY`/`JWT_KEY_ID`（`initializeJwtKeyPair`，`secret.service.ts:834-919`，RSA-2048 keypair，一律加密存 DB，不落地檔案）
  - Rotation 機制：`rotateSecret`/`rotateApiKey`/`rotateAnonKey`（`secret.service.ts:408-455`, `542-604`, `696-756`）都採 grace period 模式（舊 key 改名為 `*_OLD_<timestamp>` 並設 `expires_at`，而非直接刪除），API key 預設 24 小時、anon key 預設 168 小時（7 天，因為可能內嵌在已上架的前端/行動 App）。
- Router 層：`backend/src/api/routes/secrets/index.routes.ts` 所有 endpoint 皆掛 `verifyAdmin` middleware（如 `index.routes.ts:31,45`），並寫 audit log（`AuditService`）。

⚠️ 未驗證：`FunctionService.isSubhostingConfigured()` 的實際判斷邏輯、以及 `docker-compose.yml` 中 postgres container 啟動指令帶的 `app.encryption_key='${ENCRYPTION_KEY:-${JWT_SECRET:-dev-secret-please-change-in-production}}'`（見第 7 節）跟這裡 Node 端的 `EncryptionManager` 是否共用同一把 key／是否為兩套獨立的加密機制（Postgres 原生 pgcrypto 可能另外用於 column-level encryption，需要進一步確認）。

## 5. Cloud vs Self-hosted 分流 — `backend/src/utils/environment.ts`

```ts
export function isCloudEnvironment(): boolean {
  return !!(process.env.AWS_INSTANCE_PROFILE_NAME && process.env.AWS_INSTANCE_PROFILE_NAME.trim());
}
export function isOAuthSharedKeysAvailable(): boolean {
  return isCloudEnvironment();
}
export function getApiBaseUrl(): string {
  return process.env.API_BASE_URL || 'http://localhost:7130';
}
```
（`backend/src/utils/environment.ts` 全檔，約 30 行）

- 判斷「是否 cloud-hosted」的唯一依據是 `AWS_INSTANCE_PROFILE_NAME` 是否設定（非空字串），對應 `appConfig.cloud.instanceProfile`（`app.config.ts:146`，預設 `'insforge-instance-profile'` —— ⚠️ 這代表 `appConfig.cloud.instanceProfile` 永遠有值，但 `isCloudEnvironment()` 是直接讀 `process.env`，不是透過 `appConfig`，兩者判斷邏輯不完全一致，需留意）。
- 已知分支點：
  - `server.ts:28,292` — `if (!isCloudEnvironment()) { ... }`：只有非 cloud（self-hosted）才會註冊 root path 的重導向（例如導去 dashboard），cloud 環境略過。
  - `isOAuthSharedKeysAvailable()` 目前直接等同 `isCloudEnvironment()`，語意上是「cloud 環境可以用 insforge 官方共用的 OAuth client id/secret，不需每個 self-host 使用者自己申請 OAuth app」，⚠️ 未實際追蹤呼叫端（未在本次追蹤範圍內展開）。
- 另一種「cloud-proxied 模式」的判斷散落在別處，並非 `isCloudEnvironment()`：
  - `services.service.ts:250-276`（Compute/Fly）：以 `appConfig.fly.apiToken` 是否有值決定走 self-host Fly，否則走 cloud-mode（`PROJECT_ID`+`CLOUD_API_HOST`+`JWT_SECRET`）。
  - `backend/src/providers/database/cloud.provider.ts`：`CloudDatabaseProvider` 透過 `appConfig.cloud.apiHost`（`CLOUD_API_HOST`，預設 `https://api.insforge.dev`）向雲端控制平面要資料庫連線資訊（`connectionURL`/`parameters`），這是給「cloud-proxied」self-host 模式（資料庫本身也代管在 insforge cloud）用的 provider，跟本地 `postgres` container 是兩條不同路徑。⚠️ 未追蹤是哪個環境變數觸發選用 `CloudDatabaseProvider` vs 本地 provider（可能是 `PROJECT_ID` 存在與否，需要看 `database/base.provider.ts` 或 provider factory，未列入本次範圍）。

## 6. 前端環境變數 — `packages/dashboard/` 與 `frontend/`

- `frontend/vite.config.ts:7` `const BACKEND_URL = process.env.VITE_API_BASE_URL || 'http://localhost:7130';` — 這是 **build-time**（Node 執行 vite config 時的 `process.env`，非 `import.meta.env`）讀法，多半用來設定 dev server proxy（⚠️ 未往下確認 `BACKEND_URL` 在 config 裡實際用途，只看到宣告行）。
- Dashboard package 本身宣告的 `ImportMetaEnv` 只有一個欄位：`VITE_PUBLIC_POSTHOG_KEY?`（`packages/dashboard/src/vite-env.d.ts:1-11`），且僅在 `packages/dashboard/src/lib/analytics/posthog.tsx:4` 被讀取：`const POSTHOG_KEY = import.meta.env.VITE_PUBLIC_POSTHOG_KEY || '';`。
- `VITE_API_BASE_URL` **並沒有**在 dashboard 原始碼中透過 `import.meta.env.VITE_API_BASE_URL` 被讀取。實際的 backend URL 解析走的是「runtime config」而非「build-time env」：
  - `packages/dashboard/src/lib/config/runtime.ts:1-25` 定義 module-level 變數 `dashboardBackendUrl`，`setDashboardBackendUrl(url)` 可覆寫，`getDashboardBackendUrl()` 若未設定則 fallback 用 `window.location.origin`（`runtime.ts:16-18`），SSR 情境下回傳空字串。
  - 該 setter 由 `packages/dashboard/src/app/InsforgeDashboard.tsx:137` 呼叫：`setDashboardBackendUrl(host.backendUrl)` —— 代表 dashboard 是被「宿主」（host app，例如 `frontend/` 或 self-hosting shell）以 prop/config 注入 backend URL，而非直接讀 Vite env，這樣同一份 dashboard bundle 才能被多個宿主（cloud dashboard、self-host shell）重複使用並各自指向不同後端。
  - `getBackendUrl()`（`packages/dashboard/src/lib/utils/utils.ts:161-163`）只是 `getDashboardBackendUrl()` 的 re-export，供 MCP install 指令產生器（`connect/mcp/helpers.tsx:31,128`）、OAuth callback URL 組裝（`CustomOAuthConfigDialog.tsx:53`、`OAuthConfigDialog.tsx:31`）等處使用。
  - `utils.ts:155` `isInsForgeCloudProject`：以 `new URL(getDashboardBackendUrl()).hostname.endsWith('.insforge.app')` 判斷是否為官方雲端專案，這是**前端側**的 cloud/self-host 分流依據（跟後端 `isCloudEnvironment()` 用的 `AWS_INSTANCE_PROFILE_NAME` 是完全不同的判斷邏輯與位置，兩者未必同步 ⚠️ 未驗證是否有更上層統一判斷）。
- 因此 `VITE_API_BASE_URL`（`.env.example:55`）的實際作用範圍**很可能只在 `frontend/vite.config.ts` build/dev-server 階段**（例如設定 proxy target 或注入某個由 `host.backendUrl` 讀取的全域變數），⚠️ 未完整追蹤 `frontend/` 目錄下如何把 `BACKEND_URL` 傳給 `InsforgeDashboard` 的 `host.backendUrl` prop，需要另外查看 `frontend/src/` 入口檔案才能確認完整鏈路。

## 7. Docker Compose Env 佈線（與 `.env.example` 差異對照）

依 `docker-compose.yml`（已讀取，摘錄關鍵行）：
- `postgres` service：啟動指令帶 `-c app.encryption_key='${ENCRYPTION_KEY:-${JWT_SECRET:-dev-secret-please-change-in-production}}'`（compose 檔第 3-4 行附近），這是 **compose 層面的巢狀 fallback**：`ENCRYPTION_KEY` 未設 → 用 `JWT_SECRET` → 都未設則用寫死的 `dev-secret-please-change-in-production`。這跟第 4 節 Node 端 `EncryptionManager` 的 fallback 邏輯（`ENCRYPTION_KEY` → `JWT_SECRET` → throw）語意一致，但 compose 層多了一個「硬編碼開發用預設值」的第三層，`.env.example` 本身並未寫出這個 dev 預設值（`.env.example:73` 的 `ENCRYPTION_KEY=` 是空的，靠 compose 層的 default 撐住本機開發）。
- `insforge` service `environment:` 區塊逐一列出十幾個變數並用 `${VAR:-}` 語法從 shell/`.env` 帶入（未在 `.env` 設定時注入空字串，而不是完全不設定該 key），例如 `API_BASE_URL=${API_BASE_URL:-}`、`VITE_API_BASE_URL=${VITE_API_BASE_URL:-}`。
- `DATABASE_URL` 在 compose 中是**組合出來的**，而非直接來自 `.env.example`（`.env.example` 裡完全沒有 `DATABASE_URL` 這個變數）：
  `DATABASE_URL=postgresql://${POSTGRES_USER:-postgres}:${POSTGRES_PASSWORD:-postgres}@postgres:5432/${POSTGRES_DB:-insforge}` — host 寫死成 docker network 內的 service 名稱 `postgres`，跟 `app.config.ts:184` 讀的 `POSTGRES_HOST`（預設 `localhost`）是分開的兩個變數；compose 額外顯式覆寫 `POSTGRES_HOST=postgres`（見 compose insforge service 環境變數列表）。
- `POSTGREST_BASE_URL=http://postgrest:3000`：寫死成 docker network 內部位址（`postgrest` service 內部監聽 3000，對外映射到 `POSTGREST_PORT:-5430`），對應 `app.config.ts:190` 的預設值 `http://localhost:5430`（本機開發、非 docker 時使用）。
- `DENO_RUNTIME_URL=http://deno:7133`：同理，覆寫 `app.config.ts:217` 的 local 預設 `http://localhost:7133`。
- `ROOT_ADMIN_USERNAME`/`ROOT_ADMIN_PASSWORD` 在 compose 有雙重相容：`${ROOT_ADMIN_USERNAME:-${ADMIN_EMAIL:-admin}}`，同時把結果**又**寫回 `ADMIN_EMAIL`/`ADMIN_PASSWORD`（compose insforge service），對應 `app.config.ts:193-194` 的雙欄位 fallback（`ROOT_ADMIN_USERNAME || ADMIN_EMAIL`），這是為了相容舊版變數命名。
- `PGRST_JWT_SECRET`（postgrest service）與 `JWT_SECRET`（insforge service）共用同一個 `${JWT_SECRET:-dev-secret-please-change-in-production}`，確保 PostgREST 簽發/驗證的 JWT 跟後端一致。

## 8. Migration Bootstrap vs Baseline — `backend/src/infra/database/migrations/bootstrap/`

- `bootstrap-migrations.js`（193 行，見上方全文）：在 `node-pg-migrate` **真正跑 migration 之前**執行的前置腳本，職責有二：
  1. 相容性搬遷：把舊版 `public._migrations` ledger table 搬到 `system.migrations`（`bootstrap-migrations.js:107-140`），因為 `node-pg-migrate` 在跑任何 migration 前就會先找 ledger table，若沒搬移它會誤判成全新安裝並在 `system` schema 建一個空 ledger，導致所有 migration 被當成 pending 重跑一次。
  2. **安全防呆**（`shouldRefuseReplay`，`bootstrap-migrations.js:50-52`）：偵測「schema 已經佈建過（`auth.users`/`system.secrets`/`storage.objects` 任一存在，見 `PROVISIONED_SCHEMA_MARKERS`，`bootstrap-migrations.js:37`）但 `system.migrations` ledger 是空的」這種不一致狀態（典型情境：從備份還原資料庫、但備份沒包含 `system` schema），此時直接 `process.exit(1)` 拒絡執行，避免非幂等的 migration（例如會 rename/DROP 表的 018 號 migration）重跑造成資料庫損毀（`bootstrap-migrations.js:69-90` 的錯誤訊息詳列補救方式）。
  - 讀取 `DATABASE_URL` 環境變數建立連線（`bootstrap-migrations.js:94-101`），若未設定直接 `process.exit(1)`。
  - `package.json:27` 的 `migrate:up:local` 會先跑 `npm run migrate:bootstrap`（即本檔案）、再跑 `node-pg-migrate up`。
- `baseline-migrations.js`：⚠️ 只掃到函式簽名（`readMigrationNames`、`baselineMigrations`，第 40/48 行附近），對應 `bootstrap-migrations.js` 錯誤訊息中提到的復原路徑 `npm run migrate:baseline`（第 84-85 行註解：「Same-version restore/branch: run `npm run migrate:baseline` to stamp the ledger to match the migrations already present」）。也就是說 **bootstrap** 是「執行前置檢查 + 搬遷 ledger 表」，**baseline** 是「手動把 ledger 表『蓋章』成跟現有 migration 檔案一致而不實際重跑 SQL」，用於資料庫還原/分支等場景。未完整讀取 `baseline-migrations.js` 全文，內部細節 ⚠️ 未驗證。

## 待補充 / 未驗證清單

- `backend/src/utils/trust-proxy.ts` 的 `parseTrustProxySetting` 邏輯細節。
- `FunctionService.isSubhostingConfigured()` 判斷式。
- `isOAuthSharedKeysAvailable()` 實際呼叫端與行為。
- `frontend/` 目錄如何把 `VITE_API_BASE_URL` / `process.env.VITE_API_BASE_URL` 傳遞成 `InsforgeDashboard` 的 `host.backendUrl` prop（完整鏈路未追完）。
- Postgres container 的 `app.encryption_key` GUC 與 Node 端 `EncryptionManager` 是否共用/如何協作（是否牽涉 pgcrypto column-level encryption）。
- `database/base.provider.ts` 或 provider factory 如何依環境變數選用 `CloudDatabaseProvider` vs 本地 provider。
- `baseline-migrations.js` 完整內容。
