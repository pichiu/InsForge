# InsForge Monorepo — 進入點與初始化流程追蹤

> 產生日期：2026-07-11。此為 working document（中間工作文件），供後續 trace 使用，非最終對外文件。

## 1. Backend：`backend/src/server.ts`

### 1.1 模組載入階段（import-time side effects）
- `backend/src/server.ts:48-58`：計算 `__filename`/`__dirname`（ESM 沒有原生 `__dirname`），接著解析 `../../.env`（repo root 的 `.env`），存在則 `dotenv.config({ path: envPath })`，否則 fallback 到預設行為（look up cwd）。
- 這段在所有其他 import 副作用之後才執行（因為 import 語句在檔案最上方 line 1-47 先被 hoist 執行），但因為 dotenv 在 `createApp()` 呼叫前完成，`appConfig`（`backend/src/infra/config/app.config.js`）等模組於它們被 import 時讀到的 `process.env` 已經正確。
- 大量 route router 在檔案頂部被 import（line 8-25）：auth, database, storage, metadata, logs, docs, functions, secrets, usage, ai, memory, realtime, email, deployments, webhooks, s3-gateway, payments, advisor（後續 schedules/compute/analytics/webscraper 在 line 41-44 補充 import）。

### 1.2 `createApp()`（server.ts:60-322）— 初始化順序
1. `DatabaseManager.getInstance()` → `await dbManager.initialize()`（line 62-63，注解：create `data/app.db`）— **必須最先**，因為後續幾乎所有 service 都依賴 DB pool。
2. `StorageService.getInstance()` → `await storageService.initialize()`（line 66-67，建立 `data/storage`）。
3. `LogService.getInstance()` → `await logService.initialize()`（line 70-71，連接 CloudWatch）。
4. `await initSqlParser()`（line 74）— 初始化 SQL parser 的 WASM module（`backend/src/utils/sql-parser.ts`，底層用 `libpg-query` 的 `loadModule`/`parseSync`，for guard SQL 語句、統計 timeout 變數等）。
5. `const app = express()`（line 76）建立 Express app。
6. `app.set('trust proxy', appConfig.server.trustProxy)`（line 79）。
7. Middleware 順序（重要，會影響行為）：
   - `cors(...)`（line 82-88）：允許所有 origin（比對 Better Auth 的 `trustedOrigins: ['*']`），帶 credentials。
   - `cookieParser()`（line 89）。
   - 自訂 HTTP logging middleware（line 90-157）：monkeypatch `res.send`/`res.json` 以量測 response size，`res.on('finish', …)` 記錄 method/path/status/size/duration/ip/UA，並排除 `/logs/` 路徑避免無限迴圈。
8. **Webhook 與 S3 gateway 的 raw-body 特例（必須在 JSON middleware 之前掛載）**：
   - `app.use('/api/webhooks', express.raw({ type: 'application/json' }), webhooksRouter)`（line 161）— 保留原始 bytes 以便 signature 驗證（Stripe/Razorpay/Vercel webhook）。
   - `app.use('/storage/v1/s3', s3GatewayRouter)`（line 166）— S3 protocol gateway，完全不掛 body-parser，因為它要自己 stream body（含 `STREAMING-AWS4-HMAC-SHA256-PAYLOAD` chunked signature）。
9. 之後才掛 `express.json({ limit: jsonLimit })` 與 `express.urlencoded({ extended: true, limit: urlencodedLimit })`（line 175-176），limit 來自 `appConfig.server.maxJsonBodySize` / `maxUrlencodedBodySize`（預設高值 100mb/10mb，可由環境變數覆寫）。
10. 建立 `apiRouter = express.Router()`（line 179）。
11. JWKS handler 特別在 app 層與 apiRouter 層都掛（`/.well-known/jwks.json`，line 181-192），使用 `TokenManager.getInstance().getJwks()`。
12. `apiRouter.get('/health', …)`（line 194-202）回傳版本資訊（讀取 `../../package.json`）。
13. **Route 掛載順序**（`apiRouter.use(...)`，line 205-224）：
    auth → database → storage → metadata → logs → docs → functions → secrets → usage → ai → memory → realtime → email → deployments → schedules → payments → compute/services → analytics → webscraper → advisor。
14. `app.use('/api', apiRouter)`（line 227）— 所有上述 route 掛在 `/api` prefix 下。
15. `app.all('/functions/:slug', …)`（line 231-289）— **函式執行的相容性 proxy**：轉發請求到 Deno Deploy（Subhosting，經 `FunctionService.getInstance().isSubhostingConfigured()`/`getDeploymentUrl()`）或本地 Deno runtime（`appConfig.functions.denoRuntimeUrl`）。註解說明這是為了向後相容，之後 SDK 會直接呼叫 edge function。使用 `node-fetch` 轉發、過濾 hop-by-hop headers。
16. 非 cloud 環境時 `GET /` 導向 `/dashboard/login`（line 292-296，判斷式 `!isCloudEnvironment()`）。
17. **靜態前端服務**（line 298-316）：若 `frontendPath = path.join(__dirname, 'frontend')` 存在（即 build 後把 `frontend` dist 複製進 backend 目錄），用 `express.static(frontendPath, { index: false })` 提供靜態檔，並對 `/cloud*`、`/dashboard*` 做 SPA catch-all（`res.sendFile(index.html)`）。若不存在，則對所有未匹配路徑回傳統一格式的 404 JSON（REST 風格）。
18. `app.use(errorMiddleware)`（line 318，`backend/src/api/middlewares/error.js`）— 全域錯誤處理，掛在所有 route 之後（Express 慣例）。
19. `await seedBackend()`（line 319，`backend/src/utils/seed.ts`）— 在 app 建立完但回傳前執行 seed（見 1.4）。
20. `return app`。

### 1.3 `initializeServer()`（server.ts:327-358）— app 監聽階段
- `const app = await createApp()`。
- `app.listen(PORT, …)`（`PORT = appConfig.app.port`，預設 7130）。
- `SocketManager.getInstance().initialize(server)`（line 335-336）— Socket.IO 綁定到 HTTP server。
- `await RealtimeManager.getInstance().initialize()`（line 339-340）— pg_notify listener 啟動（見 `backend/src/infra/realtime/realtime.manager.ts:35`）。
- `FunctionService.getInstance().syncDeployment()`（line 343-348，non-blocking，`.catch` 只記 log 不擋啟動）— 同步既有 functions 到 Deno Deploy。
- `TelemetryService.getInstance().start()`（line 350）。
- 任何錯誤都會 `logger.error` 並 `process.exit(1)`（line 351-357）。
- `void initializeServer()`（line 360）— top-level 呼叫，模組載入即觸發啟動。

### 1.4 `seedBackend()`（`backend/src/utils/seed.ts`）
- 使用到的 singleton：`DatabaseManager.getInstance()`（line 17, 177）、`AuthConfigService.getInstance()`（line 26）、`OAuthConfigService.getInstance()`（line 71, 102）、`SecretService.getInstance()`（line 175）、`StripeSyncService.getInstance().seedStripeKeysFromEnv()`（line 186）、`TokenManager.getInstance().ensureKeysLoaded()`（line 260）。
- ⚠️ 未驗證：seed 的完整邏輯細節（例如是否建立預設 admin、預設 auth provider 設定等）未逐行讀取，只確認了呼叫鏈與涉及的 singleton。

### 1.5 Graceful shutdown（`cleanup()`, server.ts:362-409）
- 監聽 `SIGINT`/`SIGTERM`（line 411-412）→ `void cleanup()`。
- 依序（各自包在獨立 try/catch，任一失敗不影響其他步驟）：
  1. `RealtimeManager.getInstance().close()`。
  2. `SocketManager.getInstance().close()`。
  3. `OAuthPKCEService.getInstance().destroy()`。
  4. `TelemetryService.getInstance().stop()`。
  5. `destroyEmailCooldownInterval()`（`backend/src/api/middlewares/rate-limiters.js`，清除 email cooldown 的 interval timer）。
- 最後 `process.exit(0)`（line 408）。

### 1.6 Singleton / `getInstance()` 模式確認
專案**大量**使用 `private static instance` + `static getInstance()` 的 singleton pattern（非 DI container，未見任何 IoC container 如 InversifyJS/tsyringe）。`Grep` 在 `backend/src` 下找到 152 個檔案含 `getInstance()` 呼叫，核心確認如下：
- `DatabaseManager`（`backend/src/infra/database/database.manager.ts:19-39`）：`private static instance`, `private constructor()`, `static getInstance()`。
- `StorageService`（`backend/src/services/storage/storage.service.ts:38-75`）：同模式，`constructor` 內部呼叫 `DatabaseManager.getInstance().getPool()`（line 70）。
- `LogService`（`backend/src/services/logs/log.service.ts:23-29`）：同模式，內部依賴 `DenoSubhostingProvider.getInstance()`、`FunctionService.getInstance()`。
- `SocketManager`（`backend/src/infra/socket/socket.manager.ts:31-41`）：同模式；**特別之處**：模組頂層（line 23-25）就先呼叫 `TokenManager.getInstance()`、`SecretService.getInstance()`、`RealtimePresenceService.getInstance()` 建立依賴，並在檔案底部 `export const socketService = SocketManager.getInstance()`（line 664）額外導出一個 module-level singleton 實例。
- `RealtimeManager`（`backend/src/infra/realtime/realtime.manager.ts:21-35`）：同模式，`initialize()` 中用 `DatabaseManager.getInstance().createClient()` 建立獨立的 pg client 做 LISTEN。
- `TokenManager`（`backend/src/infra/security/token.manager.ts:46-60`）：同模式，`getJwks()` 供 server.ts 的 JWKS endpoint 使用。
- `TelemetryService`（`backend/src/services/telemetry/telemetry.service.ts:104-114`）：⚠️ 與其他不同，它是 `public constructor(...)`（line 109）而非 `private constructor`，但仍提供 `static getInstance()`（line 114）並用 `instance: TelemetryService | undefined`（line 105）—— 代表它同時允許直接 `new` 建構（可能是為了測試方便注入 mock），也支援 singleton 存取模式。⚠️ 未驗證：`getInstance()` 內部是否會 lazy-construct 或要求先手動 `new`。
- `OAuthPKCEService`（`backend/src/services/auth/oauth-pkce.service.ts:26-42`）：同標準模式。
- `FunctionService`（`backend/src/services/functions/function.service.ts:20-34`）：同標準模式，`constructor()` 中組合 `DenoSubhostingProvider.getInstance()`、`SecretService.getInstance()`，`getInstance()` 內部再組合 `DatabaseManager.getInstance()`。

**結論**：全專案未見依賴注入容器（DI container），一律用 module-level singleton（`getInstance()`）手動組裝依賴，部分 service constructor 內部直接呼叫其他 singleton 的 `getInstance()` 形成隱性依賴圖（例如 `StorageService → DatabaseManager`、`FunctionService → DenoSubhostingProvider + SecretService → DatabaseManager`）。這代表初始化順序（1.2 步驟 1-4）很重要：`DatabaseManager` 必須先 `initialize()`，否則後續 singleton 建構時透過 `getPool()`/`createClient()` 會拿到未初始化的連線。

---

## 2. Frontend self-hosting shell（`frontend/`）

### 2.1 Entry：`frontend/src/main.tsx`
- `frontend/src/main.tsx:1-2`：先 import `@insforge/dashboard/styles.css`（package 樣式）再 import 本地 `./styles.css`（順序影響 CSS override 優先權）。
- `frontend/src/main.tsx:8-14`：標準 Vite React entry，`ReactDOM.createRoot(document.getElementById('root')).render(<React.StrictMode><App /></React.StrictMode>)`。

### 2.2 `frontend/src/App.tsx`（App.tsx:1-13）
- 只有一個判斷分支：`isCloudHosting()`（`frontend/src/helpers.ts:1-7`，判斷 `window.location.origin.endsWith('.insforge.app')`）為 true 時 render `CloudHostingDashboard`（`frontend/src/cloud-hosting/CloudHostingDashboard.tsx`），否則 render `SelfHostingDashboard`（`frontend/src/self-hosting/SelfHostingDashboard.tsx`）。
- `frontend/src/helpers.ts:9-15` 另有 `isInIframe()`（判斷 `window.parent !== window`），⚠️ 未驗證用於何處（可能是 cloud-hosting 內嵌 iframe 情境判斷）。

### 2.3 `SelfHostingDashboard`（`frontend/src/self-hosting/SelfHostingDashboard.tsx:1-5`）
- 極簡：直接 `<InsForgeDashboard mode="self-hosting" />`，`InsForgeDashboard` 從 `@insforge/dashboard` package 匯入。沒有額外 backendUrl props（代表 self-hosting 模式下 dashboard 直接打自身同源 API，見下方 dashboard 內部 `normalizeBackendUrl`）。

### 2.4 `frontend/src/cloud-hosting/`
- 內含 `CloudHostingDashboard.tsx`、`useCloudHosting.ts`、`partner.service.ts`。⚠️ 未詳細展開（超出本次追蹤深度要求的核心路徑，僅確認其存在與 App.tsx 的分流關係）。這條分支會傳遞更多 host callback props（`getAuthorizationCode`、`useAuthorizationCodeRefresh` 等，見 3.1）。

---

## 3. `packages/dashboard/src/` — 可發佈的 dashboard package

### 3.1 套件對外 entry：`packages/dashboard/src/index.ts`
- `index.ts:1` 先 import `./styles.css`（package 自身樣式，供 host app 引入 `@insforge/dashboard/styles.css`）。
- 匯出核心：`InsForgeDashboard`（來自 `./app/InsforgeDashboard`，line 3）。
- 匯出導覽相關：`dashboardDeploymentsMenuItem`、`dashboardSettingsMenuItem`、`dashboardStaticMenuItems`（來自 `./navigation/menuItems`，line 4-8）及型別 `DashboardPrimaryMenuItem`（line 42）。
- 匯出大量型別（`./types`，line 9-41）：`DashboardMode`（"self-hosting" | "cloud-hosting"）、`DashboardProps`/`InsForgeDashboardProps`/`SelfHostingDashboardProps`/`CloudHostingDashboardProps`、各種 host callback 型別（backup/instance/user/metrics/advisor/posthog/apify 相關）。這是 host app（`frontend/`）與 dashboard package 之間的**唯一 contract 邊界**。

### 3.2 App factory：`packages/dashboard/src/app/InsforgeDashboard.tsx`
- `InsForgeDashboard(props: InsForgeDashboardProps)`（line 23-162）是整個 dashboard 的組裝點（app factory + provider tree），行為：
  1. 解構 host 傳入的所有 props（line 24-51），其中 `getAuthorizationCode`/`useAuthorizationCodeRefresh` 只在 `props.mode === 'cloud-hosting'` 時存在（line 52-55，type narrowing）。
  2. 用 `useMemo` 把所有 props 組成一個 `host` context 物件（line 56-93），並把 advisor 相關 callback 綁到 `advisorService`（`packages/dashboard/src/features/dashboard/services/advisor.service`，line 80-87）。
  3. `useState(() => new QueryClient({...}))`（line 124-135）建立 React Query client（`staleTime: 5min`, `gcTime: 10min`, `refetchOnWindowFocus: false`）。
  4. `setDashboardBackendUrl(host.backendUrl)`（line 137，`#lib/config/runtime`）— 把 backendUrl 寫入 runtime config 供 apiClient 使用（self-hosting 模式下 `backendUrl` 為 undefined，走同源相對路徑）。
  5. **Provider tree**（line 139-161，由外到內）：`div.insforge-dashboard` → `BrowserRouter` → `QueryClientProvider` → `DashboardHostProvider`（注入 host 物件）→ `DashboardProjectProvider`（注入 project 資訊）→ `AuthProvider` → `SocketProvider` → `ToastProvider`（來自 `@insforge/ui`）→ `PostHogAnalyticsProvider` → `SQLEditorProvider` → `AppRoutes`（`#router/AppRoutes`，實際路由渲染）。
- Router 進入點：`packages/dashboard/src/router/index.ts`、`AppRoutes.tsx`、`RequireAuth.tsx`（登入保護路由，⚠️ 未逐行讀取內容）。

---

## 4. `packages/ui/src/` — 設計系統 primitives

### 4.1 Entry：`packages/ui/src/index.ts`
- 純 barrel export 檔（無初始化邏輯、無 side-effect 程式碼），依序匯出：`Button`、`DropdownMenu*`、`Select*`、`Tooltip*`、`Switch`、`Checkbox`、`Badge`、`Input`、`SearchInput`、`InputField`、`Pagination`、`Dialog*`、`ConfirmDialog`、`MenuDialog*`、`Toast*`（含 `ToastProvider`、`useToast`、`useUploadToast`）、`CopyButton`、`CodeBlock`、`Tabs`/`Tab`、`EmptyState`、`LoadingState`、`Skeleton`，最後匯出工具函式 `cn`（`./lib`，line 102，典型 `clsx`+`tailwind-merge` 组合工具）。
- 所有元件都從 `./components` re-export，代表 `packages/ui/src/components/` 底下另有各自 index barrel（⚠️ 未逐一確認每個子元件檔案）。

### 4.2 Tailwind preset entry：`packages/ui/tailwind-preset.js`
- 位於套件 root（非 `src/` 內），`module.exports`-style 的 `tailwindPreset` 物件，`theme.extend.colors` 定義一整組 design token（`border`, `alpha-4/8/12/16`, `foreground`, `muted-foreground`, `primary`, `destructive`, `semantic-0/1/2`, `card`, `toast` 等），值全部引用 CSS variable（`var(--xxx)` 或 `rgb(var(--xxx))`），代表實際色值定義在某個全域 CSS（可能是 `packages/ui/src/styles.css`）而 tailwind preset 只是把 CSS variable 橋接成 Tailwind utility class。供 `frontend/` 與 `packages/dashboard/` 的 `tailwind.config` 透過 `presets: [tailwindPreset]` 引入。⚠️ 未驗證具體引用位置（未 grep `tailwind.config` 消費端）。

---

## 5. Deno Edge Functions Runtime（`functions/`）

### 5.1 Server entry：`functions/server.ts`
- 純 Deno script，用 `Deno.serve({ hostname, port }, handler)`（line 250-342）啟動 HTTP server。
- **啟動階段（top-level，非包在函式內，模組載入即執行）**：
  - line 5-14：解析 `PORT` env（驗證為 1-65535 的整數，否則 fallback 7133，並警告 invalid 值）。
  - line 10：`hostname = Deno.env.get('HOST') ?? '::'`（預設監聽 IPv6 all-interfaces）。
  - line 16：`console.log` 印出啟動訊息。
  - line 19：`WORKER_TIMEOUT_MS`（預設 60000ms）。
  - line 72-78：`dbConfig` 從環境變數組裝（`POSTGRES_USER/PASSWORD/DB/HOST/PORT`，預設值對應 docker-compose 中的 postgres 服務名 `postgres`）。
- **Worker template 延遲載入**：`getWorkerTemplateCode()`（line 24-30）在首次使用時才讀取 `functions/worker-template.js`（用 `Deno.readTextFile` + `import.meta.url` 算出的目錄路徑），之後快取在模組層變數 `workerTemplateCode`。
- **請求 dispatch 邏輯**（`Deno.serve` handler, line 250-342）：
  1. `GET /health`（line 255-268）：回傳 runtime/version 資訊。
  2. Slug 比對 `^\/([a-zA-Z0-9_-]+)$`（line 271，**只比對單一 path segment，不含 subpath**）：
     - `getFunctionCode(slug)`（line 81-103）：直接連 Postgres 查 `functions.definitions` table（`WHERE slug = $slug AND status = 'active'`），查不到回 404。
     - 找到後呼叫 `executeInWorker(code, req)`（line 152-248）：
       a. 取得 worker template（cache）。
       b. `getFunctionSecrets()`（line 106-149）：查 `system.secrets`（`is_active = true AND (expires_at IS NULL OR expires_at > NOW())`），用 `decryptSecret()`（line 33-69，AES-GCM，key 來自 `ENCRYPTION_KEY` 或 fallback `JWT_SECRET` 的 SHA-256 hash，需與 Node.js 端加密相容）逐一解密。
       c. 建立 `Blob` + `URL.createObjectURL` 產生 worker script URL。
       d. `new Worker(workerUrl, { type: 'module', deno: { permissions: {...} } })`（line 164-182）：**沙箱權限白名單**——`env: false`（原生 env 存取關閉，secrets 改用訊息傳遞）、`net: true`（允許對外網路，供第三方 API 呼叫）、`read/write/run/ffi/sys/import/hrtime` 全部 `false`。
       e. 設定 timeout（`WORKER_TIMEOUT_MS`，逾時 `worker.terminate()` 並回 504）。
       f. `worker.onmessage`/`worker.onerror` 處理回應或錯誤，統一 `URL.revokeObjectURL` 清理。
       g. `worker.postMessage({ code, requestData, secrets })`（line 246）把 function 原始碼、request 資料（url/method/headers/body）、解密後 secrets 一次性丟進 worker。
  3. `GET /info`（line 323-338）：回傳 runtime/env/database host 資訊（不含密碼）。
  4. 其餘 404。
- 每次 invocation 完成會印出 JSON 格式的 log（含 timestamp/level/slug/method/status/duration），失敗則印 error log（line 291-319）。

### 5.2 `functions/worker-template.js`（沙箱內程式碼）
- 檔案開頭（line 1-8）註解說明：跑在 Deno Worker 環境中，每個 worker 是「一次性」（fresh worker per request, single execution then terminate）。
- **Security blackout（top-level, line 9-52）**：在任何 import 執行「前」（用 top-level await 語意上的最早時機）以 `Object.defineProperty(globalThis, 'Deno', { value: Object.freeze({env: mockDenoEnv}), configurable:false, writable:false })` **鎖死** `Deno.env`，防止使用者 function code 或其依賴（如 `debug` package）意外觸發 `NotCapable` 錯誤或存取真實環境變數；同時 shadow `process.env` 為 sterile object（僅 `NODE_ENV: production`）。任何 shadow 失敗會直接 `self.postMessage({success:false, error:'Security Initialization Error', status:500})` 並 `self.close()` 中止 worker。
- **Early message buffering（line 54+ 起，⚠️ 未完整讀取全文）**：註解提及因為 `server.ts` 的 `executeInWorker` 在 `new Worker()` 後立即呼叫 `postMessage`，而 Web Worker 不會 queue 在 `onmessage` handler 註冊前抵達的訊息，因此需要提早緩衝訊息，避免 race condition 遺失第一則 postMessage。⚠️ 未驗證後續完整實作細節（緩衝機制、function code 實際 eval/執行方式、response 序列化細節），只確認了開頭的兩個關鍵段落。

### 5.3 `functions/deno.json`
- ⚠️ 未展開內容，僅確認檔案存在（Deno 專案設定檔，通常含 import map / compiler options）。

---

## 6. CLI / Script 進入點

### 6.1 `backend/scripts/`
- `check-migration-duplicates.js`：由 `backend/package.json:25` 的 `migrate:check-duplicates` script 呼叫（`node scripts/check-migration-duplicates.js`）。⚠️ 未展開內容，用途應為檢查 migration 檔名/版本號重複。
- `test-deno-subhosting.sh`：⚠️ 未展開，推測為手動測試 Deno Subhosting provider 整合的 shell script。

### 6.2 Root `scripts/`
- `sync-skills.sh`、`update-mintlify-skill.sh`：與本 repo 的 Claude Code skills / Mintlify 文件同步相關，非 runtime 進入點，屬於 repo 維護工具鏈。⚠️ 未展開內容。

### 6.3 Migration bootstrap（`backend/src/infra/database/migrations/bootstrap/`）
- **`bootstrap-migrations.js`**（對應 `backend/package.json:20-22`：`migrate:bootstrap` → `migrate:up` 的前置步驟）：
  - 匯出 `bootstrapMigrations()`（line 92-180）與純函式 `shouldRefuseReplay({ledgerTableExists, ledgerRowCount, schemaProvisioned})`（line 50-52，特別標註為方便單元測試而導出）。
  - 目的：在 `node-pg-migrate` 執行前，把舊的 `public._migrations` table 搬到 `system.migrations`（line 123-140，因為 node-pg-migrate 在跑 migration 前就會先找 tracking table，若在 migration 檔內搬移會太晚）。
  - **安全防呆機制**（line 141-158）：若 `system.migrations` 已存在但是空的（`ledgerRowCount === 0`），且 schema 已經被 provision 過（透過檢查 `auth.users`/`system.secrets`/`storage.objects` 是否存在，`PROVISIONED_SCHEMA_MARKERS`，line 37, 54-62），則判定是「從 backup 還原但漏了 system.migrations」的異常狀態，**拒絕**讓 node-pg-migrate 重跑（因為許多 migration 非幂等，例如 018 會 move/rename table），改為 `logger.error` 印出詳細修復指引並 `process.exit(1)`（line 156-157，`logInconsistentLedgerError()` line 69-90）。
  - 全新安裝（old/new table 都不存在）：只建立 `system` schema（line 162-166）讓後續 node-pg-migrate 自己建 tracking table。
  - Line 183-191：`if (!process.env.VITEST) { bootstrapMigrations().catch(...) }` — **auto-run as script**，但在 unit test（VITEST env）環境下不會自動執行，改由測試自行 import 呼叫。
  - 用 `tsx` 執行（見 `backend/package.json:20` 註解說明，line 25-27 of the .js file 也有註解提及）。
- **`baseline-migrations.js`**：對應 `migrate:baseline` script（`backend/package.json:21`），⚠️ 未展開內容，依 `bootstrap-migrations.js` 註解（line 84-85）推測其用途是「把 ledger 標記為與現有 migration 檔案一致」，用於 same-version restore/branch 的修復流程。

### 6.4 `backend/package.json` migrate 相關 scripts（line 20-28）
```
migrate:bootstrap    → tsx bootstrap-migrations.js
migrate:baseline     → tsx baseline-migrations.js
migrate:up           → migrate:bootstrap && node-pg-migrate up (schema=system, table=migrations)
migrate:down         → node-pg-migrate down
migrate:create       → node-pg-migrate create
migrate:check-duplicates → node scripts/check-migration-duplicates.js
migrate:redo         → migrate:bootstrap && node-pg-migrate redo
migrate:up:local / migrate:down:local → 同上但用 dotenv-cli 載入 ../.env（本地開發用）
```

### 6.5 Root `package.json` scripts（line 12-30）
- `dev` → `turbo run dev`（透過 Turborepo 併行跑各 package 的 dev script）。
- `dev:debug` → `concurrently` 同時跑 `dev:backend:debug` 與 `dev:frontend:debug`（各自用 `cross-env DEBUG_MODE=true` / `VITE_DEBUG_MODE=true`）。
- `build` → `turbo run build`；`start` → `cd backend && npm run start`；`start:prod` → `build && start`。
- `install:all` → 根目錄、backend、frontend 各自 `npm install`（三個獨立 node_modules，非純 workspace hoist）。

---

## 7. Docker Compose 啟動指令鏈（已知資訊，僅引用不重新推導）

- 已知指令鏈：`npm install && turbo build ... && migrate:up && concurrently backend+frontend dev`。
- 相關檔案（存在但本次未逐行展開）：`docker-compose.yml`、`docker-compose.override.yml`、`docker-compose.prod.yml`、`docker-compose.dokploy.yml`（皆位於 repo root）。⚠️ 未驗證各檔案間的具體 override 關係與各自 command 是否與上述指令鏈完全一致，僅確認檔案存在且指令鏈與 `migrate:up`（6.4）、`dev`（6.5）script 定義吻合。

---

## 8. 尚待深入（本次未展開，供後續 trace 參考）

- `packages/dashboard/src/router/AppRoutes.tsx`、`RequireAuth.tsx` 的完整路由表與登入保護邏輯。
- `frontend/src/cloud-hosting/CloudHostingDashboard.tsx`、`useCloudHosting.ts`、`partner.service.ts` 的完整實作（partner iframe 通訊協定等）。
- `functions/worker-template.js` 完整內容（message buffering 之後：實際如何 `eval`/`import` 使用者 function code、如何組裝 `Request`/`Response`、如何把結果 postMessage 回主 thread）。
- `backend/src/utils/seed.ts` 完整 seed 邏輯（各 config service 的 seed 細節、是否建立預設帳號）。
- `backend/scripts/check-migration-duplicates.js`、`test-deno-subhosting.sh`、root `scripts/*.sh` 的具體實作。
- `backend/src/infra/database/migrations/bootstrap/baseline-migrations.js` 完整邏輯。
- Docker Compose 各檔案的完整 service 定義與 command override 細節。
