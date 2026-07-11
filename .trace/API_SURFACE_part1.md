# InsForge API 介面參考文件（Part 1／2）

> 產生日期：2026-07-11。來源：對 `backend/src/api/routes/` 原始碼、`packages/shared-schemas/src/` Zod schema、`backend/src/api/middlewares/` 的直接讀取，交叉參照 `.trace/_context/recon.md`、`entry_points.md`、`integrations.md`。
>
> **Part 1 內容**：整體架構、Authentication & Authorization 模型、Error handling、Rate limiting、PostgREST 直通層、`auth`／`database`／`storage`／`metadata` 四個核心 domain 的完整介面清單。
> **Part 2 內容**（見 `API_SURFACE_part2.md`）：其餘 18 個 domain（logs/docs/functions/secrets/usage/ai/memory/realtime/email/deployments/schedules/payments/compute/analytics/webscraper/webhooks/s3-gateway/advisor）。

---

## 0. 整體架構

InsForge backend 是一個 Express.js monolith，所有 domain route 都掛載在 `apiRouter`（`backend/src/server.ts:179`），再以 `app.use('/api', apiRouter)`（`server.ts:227`）掛在 `/api` prefix 下。因此本文件所有 path 若無特別註明，實際請求 URL 一律是 `<backend-origin>/api/<domain>/...`。

route 群組掛載順序（`server.ts:205-224`，共 22 個）：

```
auth → database → storage → metadata → logs → docs → functions → secrets →
usage → ai → memory → realtime → email → deployments → schedules → payments →
compute/services → analytics → webscraper → advisor
```

（`webhooks` 與 `s3-gateway` 兩個 domain **不在** `/api` prefix 下，而是在 `apiRouter` 掛載之前，於 app 層直接掛在 `/api/webhooks` 與 `/storage/v1/s3`，因為它們需要 raw body / 自訂 SigV4 簽章，見 Part 2。)

除了 app 層 REST API，InsForge 還內建一個**獨立的 PostgREST 直通層**（見第 4 節），與 app 層的 `/api/database/records`、`/api/database/rpc` 在底層共用同一個 PostgREST 實例。

分層慣例（`.trace/_context/recon.md` 摘要，`insforge-dev` skill 定義）：route → service → provider/infra，成功回應通常是 raw JSON（不包 `{ data }` 外層），驗證用共用 Zod schema（`packages/shared-schemas/`）+ `AppError`。

---

## 1. Authentication & Authorization 模型

所有認證邏輯集中於 `backend/src/api/middlewares/auth.ts`。核心設計：**所有 credential 都走 `Authorization` header（Bearer scheme），依「credential 形狀」分派，不做失敗後降級（fail closed）**。

### 1.1 Credential 種類與分派規則（`auth.ts:79-107` 註解）

| Credential 前綴/形狀 | 對應角色 | 說明 |
|---|---|---|
| `Bearer ik_...`（或 `x-api-key` header，向後相容） | admin API key | 專案層級的 admin 憑證，繞過使用者身份 |
| `Bearer anon_...` | `anon` role | 未登入用戶使用的 opaque anon key；`sub` claim 不帶入 DB，`auth.uid()` 為 NULL |
| 其他任意 Bearer token | JWT（依 `role` claim 決定 `admin`/`authenticated`/legacy `anon`） | 使用者登入後取得的 JWT |
| 缺少 credential | 一律 401 | 不會靜默降級為 anon |

### 1.2 五種 middleware（`auth.ts`）

| Middleware | 定義位置 | 接受的憑證 | 典型使用場景 |
|---|---|---|---|
| `verifyUser` | `auth.ts:93-107` | API key、anon key、任意 JWT（依 shape 分派給 `verifyApiKey`/`verifyAnonKey`/`verifyToken`） | 一般使用者可存取的 client-facing endpoint（如 `/database/records`、`/storage` 物件操作） |
| `verifyAdmin` | `auth.ts:112-157` | API key，或 `role === 'project_admin'` 的 JWT；否則丟 403 `AUTH_UNAUTHORIZED` | Dashboard／管理性操作（建表、改設定、看所有使用者等），幾乎所有 admin-only domain 的預設守門 |
| `verifyApiKey` | `auth.ts:163-192` | 僅 `ik_...` API key（Bearer 或 `x-api-key`） | 需要嚴格限定「僅限伺服端持有的專案 API key」的路由，如 MCP usage 回報 |
| `verifyAnonKey` | `auth.ts:201-229` | 僅 `anon_...` opaque key | 匿名存取場景，映射到 `anon` role；安全邊界交由 Postgres RLS policy 負責 |
| `verifyToken` | `auth.ts:235-278` | 僅 JWT，任意 role，設定 `req.user` | 需要「已登入」但不區分 admin/user 的場景，如 `/auth/profiles/current` |
| `verifyCloudBackend` | `auth.ts:284-318` | 用 JWKS 驗證來自 `api.insforge.dev` 的 JWT，檢查 `project_id` claim，設定 `req.projectId` | InsForge Cloud 後端對單一 project 的 backend-to-backend 呼叫（如 `usage/stats`），非一般客戶端可用 |

`UserContext`（`auth.ts:8-19`）型別的 `id` 欄位語意特別：對已登入使用者是 UUID，對 admin 是 admin id，對 anon 固定字串 `'anonymous'`。**只有 `role === 'authenticated'` 的 UUID 會進入 DB 層的 row-ownership 判斷**（claims、owner 欄位），admin/anon 兩種 label 不會被誤當成 uuid-typed claim 使用。

### 1.3 各 domain 認證慣例小結

- **完全公開（無認證）**：`GET /auth/sessions`（登入）、`POST /auth/users`（實際仍走 `verifyUser`，見下）之外的多數 email/OAuth 流程（因為呼叫者本來就還沒有身份）、`docs` domain 全部端點、`webhooks`（改以簽章驗證取代）。
- **`verifyUser`**：資料操作類 client SDK 端點（`database/records`、`database/rpc`、`storage` 物件上傳下載、`ai` 推論呼叫）。
- **`verifyAdmin`**：幾乎所有 dashboard 管理端點（建表/改 schema、bucket 設定、OAuth 設定、secrets、realtime channel 管理、compute、payments 設定與 catalog、advisor 等）。
- **`verifyApiKey`（單獨使用）**：`memory` domain 整體（`router.use(verifyApiKey)`）、`usage/mcp` 上報。
- **`verifyCloudBackend`**：`usage/stats`（cloud billing 查詢）。
- **自訂簽章機制（非 `verify*` 家族）**：`webhooks/*`（Stripe/Razorpay/Vercel 各自的 HMAC 簽章驗證）、`s3-gateway`（AWS SigV4，`s3Sigv4Middleware`，見 Part 2）。

---

## 2. Error Handling Pattern

### 2.1 `AppError`（`backend/src/utils/errors.ts:5-15`）

```ts
export class AppError extends Error {
  constructor(
    public message: string,
    public statusCode: number = 500,
    public code: string,
    public nextActions?: string
  ) { super(message); this.name = 'AppError'; }
}
```

所有 route handler 一律 `throw new AppError(message, statusCode, ERROR_CODES.XXX, nextActions)` 或 `next(error)`。`code` 欄位的合法值集中定義在 `packages/shared-schemas/src/error-codes.schema.ts`，依 domain 分組，例如：

```
AUTH_INVALID_EMAIL / AUTH_WEAK_PASSWORD / AUTH_INVALID_CREDENTIALS / AUTH_INVALID_API_KEY /
AUTH_EMAIL_EXISTS / AUTH_USER_NOT_FOUND / AUTH_UNAUTHORIZED / AUTH_NEED_VERIFICATION / AUTH_SIGNUP_DISABLED
DATABASE_INVALID_PARAMETER / DATABASE_VALIDATION_ERROR / DATABASE_CONSTRAINT_VIOLATION /
DATABASE_NOT_FOUND / DATABASE_DUPLICATE / DATABASE_PERMISSION_DENIED / DATABASE_FORBIDDEN
STORAGE_ALREADY_EXISTS / STORAGE_INVALID_FILE_TYPE / STORAGE_INSUFFICIENT_QUOTA / STORAGE_NOT_FOUND
REALTIME_CHANNEL_NOT_FOUND / REALTIME_UNAUTHORIZED / ...
AI_INVALID_API_KEY / AI_INVALID_MODEL / AI_UPSTREAM_UNAVAILABLE
ANALYTICS_NOT_CONNECTED / ANALYTICS_UNAVAILABLE
LOGS_AWS_NOT_CONFIGURED / LOG_NOT_FOUND
COMPUTE_CLOUD_UNAVAILABLE / ...
```
（`error-codes.schema.ts:7-64`，⚠️ 未逐一列出完整清單，僅節錄代表性 domain。）

### 2.2 `errorMiddleware`（`backend/src/api/middlewares/error.ts:8-66`）

全域錯誤處理 middleware，掛在所有 route 之後（`server.ts:318`）。依錯誤型別分流，統一轉成同一種 JSON 回應格式：

1. **`AppError` 實例**（`error.ts:18-20`）→ 直接用其 `code`/`message`/`statusCode`/`nextActions` 組回應；`statusCode === 401` 的錯誤不記 log（避免每次未登入請求都洗 log）。
2. **`SyntaxError`（JSON.parse 失敗）**（`error.ts:23-31`）→ 400 `INVALID_INPUT`。
3. **`pg` 的 `DatabaseError`**（`error.ts:34-45`）→ 透過 `getDatabaseErrorDetails()`（`utils/errors.ts:43+`）依 Postgres error code 對照表轉換，例如：
   - `23505`（unique violation）→ 409 `DATABASE_DUPLICATE`，並從 `detail` 解析出衝突欄位名放進 `nextActions`。
   - `23503`（FK violation）→ 400 `DATABASE_CONSTRAINT_VIOLATION`。
   - `23502`（NOT NULL violation）→ 400 `MISSING_FIELD`。
   - `42P01`（表不存在）→ 400 `DATABASE_VALIDATION_ERROR`。
4. **body-parser 的 `entity.parse.failed`**（`error.ts:53-61`）→ 400 `INVALID_INPUT`。
5. **其餘未知錯誤** → 500 `INTERNAL_ERROR`。

統一回應格式（`ErrorResponse`，`backend/src/utils/response.ts:6-11`）：

```json
{
  "error": "DATABASE_DUPLICATE",
  "message": "duplicate key value violates unique constraint \"users_email_key\"",
  "statusCode": 409,
  "nextActions": "The value for 'email' must be unique. Try a different value."
}
```

`nextActions` 是 InsForge 特有欄位（`backend/src/utils/next-actions.ts`），提供給呼叫端（尤其是 coding agent）具體的修復建議文字，而非只回一句錯誤訊息——這與「面向 agent 使用」的產品定位一致（見 `recon.md` 第 1 節）。

---

## 3. Rate Limiting 機制摘要

引用自 `.trace/_context/integrations.md` 第 11 節，實作於 `backend/src/api/middlewares/rate-limiters.ts`（`express-rate-limit`）：

| 限流器 | 窗口 | 上限 | 適用範圍 |
|---|---|---|---|
| `sendEmailOTPRateLimiter` | 15 分鐘 | 每 IP 5 次 | `POST /auth/email/send-verification`、`/send-reset-password` |
| `verifyOTPRateLimiter` | 15 分鐘 | 每 IP 10 次 | `POST /auth/email/verify`、`/exchange-reset-password-token`、`/reset-password` |
| `s3AccessKeyManagementRateLimiter` | 15 分鐘 | 每 IP 20 次 | `storage/s3/access-keys` 系列 |
| `computeLogsRateLimiter` | 1 分鐘 | 每 IP 120 次 | `compute/services/:id/logs` |
| 寫入端點動態限流器（`functionsWriteLimiter`/`deploymentsWriteLimiter`/`computeWriteLimiter` 等） | 5 分鐘 | 依 `getWriteEndpointLimit(category)` 動態決定 | functions/deployments/compute 等 domain 的寫入操作，可整體透過 `isWriteRateLimitDisabled()` 關閉 |

未見任何 domain 使用 circuit breaker；rate limiting 純粹是 Express middleware 層級的 IP-based 限流，不是分散式（多實例部署下各實例各自計數，⚠️ 未驗證是否有共享 store）。

---

## 4. PostgREST 直通層（另一個 API Surface）

InsForge 在同一個 docker-compose 拓樸中，除了 app 層 Express API，還獨立跑一個 **PostgREST** 容器（`docker-compose.yml:26-42`，image `postgrest/postgrest:v12.2.12`，對外預設 port `5430`），這是**第二個、與 app API 平行存在的 API surface**：

- **自動生成**：PostgREST 直接依 Postgres `public` schema（`PGRST_DB_SCHEMA: public`）反射生成 REST endpoint，每個 table/view 自動對應一組 CRUD 路徑（`GET/POST/PATCH/DELETE /<table>`），無需額外程式碼；新增一個 table 後立即可透過 PostgREST 存取，不需要重啟或重新部署 app。
- **安全邊界＝Row Level Security**：PostgREST 依請求所帶的 JWT（`PGRST_JWT_SECRET` 與 app 共用同一把 `JWT_SECRET`）解析出 Postgres role/claims，實際資料可見範圍完全由 **Postgres RLS policy** 決定，而非應用層程式碼做權限判斷。`PGRST_DB_ANON_ROLE: anon` 定義了未帶身份 claim 時採用的資料庫角色。
- **與 app API 的關係——並非各自獨立實作，而是共用同一底層**：`backend/src/api/routes/database/records.routes.ts`（`/api/database/records/:tableName`）與 `rpc.routes.ts`（`/api/database/rpc/:functionName`）**本身就是把請求代理轉發給這個 PostgREST 實例**（`PostgrestProxyService`，`backend/src/services/database/postgrest-proxy.service.ts`），而非重新實作一套 CRUD 邏輯。差異在於：
  - App 層先跑 `verifyUser` 做 InsForge 自訂的憑證解析（API key / anon key / JWT），再依角色分流呼叫 `forwardAsAdmin()` / `forwardAsUser()` / `forwardAsAnon()`（`postgrest-proxy.service.ts:100-152`）把對應的 Postgres role 塞進轉發請求，讓 PostgREST 端仍然套用 RLS。
  - App 層額外做 table 名稱驗證、`schema=` query 參數轉譯成 PostgREST 的 `Accept-Profile`/`Content-Profile` header（`resolvePostgrestSchema()`）、寫入操作後透過 Socket.IO 廣播 `DATA_UPDATE` realtime 事件（`records.routes.ts:121-132`）。
  - 直接打 PostgREST 原生 port（5430）則完全繞過這些加值邏輯（不會觸發 realtime 廣播、也不受 InsForge 自訂錯誤格式包裝，而是 PostgREST 原生錯誤格式），但換取更完整的 PostgREST 查詢語法（embedding、`or=`、`select` 巢狀等）與更低延遲。
- **`/api/database/records` 也支援 PostgREST 風格 query 語法**：即使走 app 層代理，查詢參數仍是 PostgREST 慣例（`?status=eq.active&order=createdAt.desc&select=id,title`），可見 `openapi/records.yaml`。

---

## 5. `auth` domain — `/api/auth`（完整清單）

掛載結構（`backend/src/api/routes/auth/index.routes.ts:80-82`）：`index.routes.ts` 本體 + 子路由 `/admin`（`admin.routes.ts`，dashboard 管理員登入）、`/oauth`（`oauth.routes.ts`，社交登入）、`/oauth/custom`（`custom-oauth.routes.ts`，自訂 OIDC）。

### 5.1 `index.routes.ts`（掛載於 `/api/auth`）

| Method | Path | 認證 | Rate limiter | 說明 |
|---|---|---|---|---|
| POST | `/users` | `verifyUser` | 無 | **註冊（signup）**——已登入呼叫者（admin/api-key）可視為代建使用者 |
| POST | `/sessions` | 無 | 無 | **登入（login）**，email+password |
| POST | `/id-token` | 無 | 無 | 用第三方 ID token 登入（目前僅 `google`） |
| POST | `/refresh` | 無（走 cookie 或 body 內 refreshToken） | 無 | 刷新 access token |
| POST | `/logout` | 無 | 無 | 登出，清除 refresh cookie |
| GET | `/sessions/current` | `verifyToken` | 無 | 取得目前登入使用者 session |
| GET | `/public-config` | 無 | 無 | 公開安全子集的 auth 設定 |
| PATCH | `/profiles/current` | `verifyToken` | 無 | 更新目前使用者 profile JSON |
| GET | `/profiles/:userId` | 無 | 無 | 取得使用者公開 profile |
| GET / PUT | `/config` | `verifyAdmin` | 無 | 讀取/更新完整 auth 設定（admin） |
| GET | `/users` | `verifyAdmin` | 無 | 列出所有使用者（分頁、搜尋） |
| GET | `/users/:userId` | `verifyAdmin` | 無 | 取得單一使用者 |
| DELETE | `/users` | `verifyAdmin` | 無 | 批次刪除使用者 |
| POST | `/tokens/anon` | `verifyAdmin` | 無 | **已棄用**：取得 opaque anon key（改用 `GET /metadata/anon-key`） |
| POST | `/email/send-verification` | 無 | `sendEmailOTPRateLimiter` | 寄送驗證信（依設定為 code 或 link） |
| POST | `/email/verify` | 無 | `verifyOTPRateLimiter` | 用 6 碼 OTP 驗證信箱 |
| GET | `/email/verify-link` | 無 | 無 | 瀏覽器點擊連結驗證（redirect） |
| POST | `/email/send-reset-password` | 無 | `sendEmailOTPRateLimiter` | 寄送重設密碼信 |
| POST | `/email/exchange-reset-password-token` | 無 | `verifyOTPRateLimiter` | 用 OTP 換取 reset token |
| POST | `/email/reset-password` | 無 | `verifyOTPRateLimiter` | 用 token 提交新密碼 |
| GET | `/email/reset-password-link` | 無 | 無 | 瀏覽器點擊連結驗證重設請求 |
| GET / PUT | `/smtp-config` | `verifyAdmin` | 無 | 讀取/更新 SMTP 設定 |
| GET | `/email-templates` | `verifyAdmin` | 無 | 列出 email 模板 |
| PUT | `/email-templates/:type` | `verifyAdmin` | 無 | 更新單一 email 模板 |

### 5.2 `admin.routes.ts`（掛載於 `/api/auth/admin`，Dashboard 管理員登入，與一般使用者登入分開的獨立體系）

| Method | Path | 認證 | 說明 |
|---|---|---|---|
| POST | `/sessions` | 無 | Dashboard 管理員登入（username/password） |
| POST | `/sessions/exchange` | 無 | 用授權碼交換 dashboard admin session |
| GET | `/sessions/current` | `verifyToken`（另檢查 `role === 'project_admin'`） | 取得目前 admin session |
| POST | `/refresh` | 無（cookie-based） | 刷新 admin access token |
| POST | `/logout` | 無 | 登出 admin session |

### 5.3 `oauth.routes.ts`（掛載於 `/api/auth/oauth`，第三方社交登入）

| Method | Path | 認證 | 說明 |
|---|---|---|---|
| GET | `/configs` | `verifyAdmin` | 列出各 provider 的 OAuth 設定 |
| GET / PUT / DELETE | `/:provider/config` | `verifyAdmin` | 單一 provider 設定 CRUD |
| POST | `/configs` | `verifyAdmin` | 新增 OAuth 設定 |
| GET | `/:provider` | 無 | 啟動 OAuth 流程（回傳 `authUrl`，PKCE） |
| GET / POST | `/:provider/callback` | 無 | Provider callback（POST 供 Apple form_post 用） |
| GET | `/shared/callback/:state` | 無 | Cloud 共用 OAuth key 的共用 callback |
| POST | `/exchange` | 無 | 用 PKCE code 交換 access/refresh token |

### 5.4 `custom-oauth.routes.ts`（掛載於 `/api/auth/oauth/custom`，自訂 OIDC provider）

結構與 5.3 對稱：`GET/POST/PUT/DELETE /configs`、`/:key/config`（`verifyAdmin`）+ `GET /:key`、`/:key/callback`（無認證，OIDC 流程本身）。

### 5.5 Request/Response 範例

**簽名／型別來源**：`packages/shared-schemas/src/auth-api.schema.ts`、`auth.schema.ts`。

#### 5.5.1 `POST /api/auth/users`（Signup）

Request（`createUserRequestSchema`，`auth-api.schema.ts:36-42`）：

```json
{
  "email": "dev@example.com",
  "password": "s3cure-Pass!",
  "name": "Dev User",
  "redirectTo": "https://myapp.com/welcome",
  "autoConfirm": false
}
```

Response（`createUserResponseSchema`，`auth-api.schema.ts:154-160`）：

```json
{
  "user": {
    "id": "b3f1...-uuid",
    "email": "dev@example.com",
    "emailVerified": false,
    "providers": ["email"],
    "createdAt": "2026-07-11T02:00:00.000Z",
    "updatedAt": "2026-07-11T02:00:00.000Z",
    "profile": { "name": "Dev User" },
    "metadata": null
  },
  "accessToken": null,
  "requireEmailVerification": true,
  "csrfToken": null
}
```

註：`accessToken` 在需要 email 驗證前為 `null`；Web client 透過 httpOnly cookie 拿 refresh token 並回傳 `csrfToken`，Mobile/desktop client（依 `client_type` query）改在 body 中拿到 `refreshToken` 欄位。

#### 5.5.2 `POST /api/auth/sessions`（Login）

Request（`createSessionRequestSchema`，`auth-api.schema.ts:47-50`）：

```json
{ "email": "dev@example.com", "password": "s3cure-Pass!" }
```

Response（`createSessionResponseSchema`，`auth-api.schema.ts:166-171`）：

```json
{
  "user": {
    "id": "b3f1...-uuid",
    "email": "dev@example.com",
    "emailVerified": true,
    "providers": ["email"],
    "createdAt": "2026-07-11T02:00:00.000Z",
    "updatedAt": "2026-07-11T02:00:00.000Z",
    "profile": { "name": "Dev User" },
    "metadata": null
  },
  "accessToken": "eyJhbGciOi...",
  "csrfToken": "a1b2c3..."
}
```

`userSchema`（`auth.schema.ts:52-61`）欄位型別：`id: uuid`, `email: string`, `emailVerified: boolean`, `providers?: string[]`, `createdAt/updatedAt: string(ISO)`, `profile: { name?, avatar_url?, ...passthrough } | null`, `metadata: Record<string, unknown> | null`。

---

## 6. `database` domain — `/api/database`（完整清單，最核心 domain）

子路由掛載（`backend/src/api/routes/database/index.routes.ts:20-28`）：`/tables`、`/records`、`/rpc`、`/advance`、`/migrations`、`/backups`（僅 self-host，`isCloudEnvironment()` 為 false 時掛載）、`/admin`。

### 6.1 `index.routes.ts` 直掛路由（`/api/database`）

| Method | Path | 認證 | 說明 |
|---|---|---|---|
| GET | `/schemas` | `verifyAdmin` | 列出資料庫 schema |
| GET | `/functions` | `verifyAdmin` | 列出 Postgres function（`?schema=`） |
| GET | `/indexes` | `verifyAdmin` | 列出索引 |
| GET | `/policies` | `verifyAdmin` | 列出 RLS policy |
| GET | `/triggers` | `verifyAdmin` | 列出 trigger |

### 6.2 `/tables`（`tables.routes.ts`，DDL：建表、改 schema）

| Method | Path | 認證 | 說明 |
|---|---|---|---|
| GET | `/` | `verifyAdmin` | 列出所有 table（`?schema=`） |
| POST | `/` | `verifyAdmin` | 建立新 table |
| GET | `/:tableName/schema` | `verifyAdmin` | 取得單一 table schema（欄位、FK） |
| PATCH | `/:tableName/schema` | `verifyAdmin` | 改 schema（增刪欄位/FK、改名） |
| DELETE | `/:tableName` | `verifyAdmin` | 刪除 table |

### 6.3 `/records`（`records.routes.ts`，DML：CRUD 資料——**PostgREST 代理**，見第 4 節）

| Method | Path | 認證 | 說明 |
|---|---|---|---|
| ALL（GET/POST/PATCH/PUT/DELETE） | `/:tableName` | `verifyUser` | 轉發到 PostgREST（PostgREST filter/select 語法） |
| ALL | `/:tableName/*path` | `verifyUser` | 轉發巢狀/embed 路徑 |

依 caller role 分流：`project_admin`/API key → `forwardAsAdmin()`；`authenticated`（已登入非 anon） → `forwardAsUser()`（RLS 生效）；anon/未登入 → `forwardAsAnon()`。POST/DELETE 成功後透過 `SocketManager` 廣播 `DATA_UPDATE` realtime 事件（`records.routes.ts:121-132`）。

### 6.4 `/rpc`（`rpc.routes.ts`，呼叫 Postgres function）

| Method | Path | 認證 | 說明 |
|---|---|---|---|
| ALL | `/:functionName` | `verifyUser` | 轉發到 PostgREST `/rpc/:functionName` |

### 6.5 `/advance`（`advance.routes.ts`，進階／危險操作，皆 `verifyAdmin`）

| Method | Path | 說明 |
|---|---|---|
| POST | `/rawsql/unrestricted` | 以 DB owner／root 權限執行任意 SQL |
| POST | `/rawsql` | 以 `project_admin` 權限執行 SQL |
| POST | `/export` | 匯出資料庫（`sql` 或 `json` 格式） |
| POST | `/bulk-upsert` | 上傳 CSV/JSON（`multipart/form-data`）批次 upsert |
| POST | `/import` | 上傳 SQL 檔匯入資料庫 |

### 6.6 `/migrations`、`/backups`

- `/migrations`：`GET /`（列出）、`POST /`（建立自訂 migration：`version`/`name`/`sql`），皆 `verifyAdmin`。
- `/backups`（僅 self-host）：`GET /`、`POST /`、`PATCH /:id`（改名）、`DELETE /:id`、`POST /:id/restore`，皆 `verifyAdmin`。

### 6.7 `/admin`（`admin.routes.ts`，Dashboard 專用的記錄 CRUD，`router.use(verifyAdmin)` 全域套用）

| Method | Path | 說明 |
|---|---|---|
| GET | `/tables/:tableName/records` | 分頁/搜尋/排序/篩選列出記錄 |
| GET | `/tables/:tableName/records/lookup` | 依欄位值查單筆記錄 |
| POST | `/tables/:tableName/records` | 批次建立（陣列 body） |
| PATCH | `/tables/:tableName/records` | 依 `pkKeys` 更新單筆 |
| DELETE | `/tables/:tableName/records` | 依 `pkKeys` 陣列批次刪除 |

這條路徑與 `/records`（6.3）的差異：`/admin` 是**直接操作**（非 PostgREST 代理），繞過使用者層 RLS（因為呼叫者本身已是 admin），供 Dashboard 資料瀏覽器使用；`/records` 才是給一般 client SDK 用、走 RLS 的資料存取路徑。

### 6.8 Request/Response 範例

型別來源：`packages/shared-schemas/src/database-api.schema.ts`、`database.schema.ts`。

#### 6.8.1 `POST /api/database/tables`（建立 table）

Request（`createTableRequestSchema`，`database-api.schema.ts:15-23`，衍生自 `tableSchema`）：

```json
{
  "tableName": "posts",
  "columns": [
    { "columnName": "id", "type": "uuid", "isPrimaryKey": true, "isNullable": false, "isUnique": true, "defaultValue": "gen_random_uuid()" },
    { "columnName": "title", "type": "string", "isNullable": false, "isUnique": false },
    { "columnName": "author_id", "type": "uuid", "isNullable": false, "isUnique": false }
  ],
  "foreignKeys": [
    {
      "referenceTable": "users",
      "referenceColumns": [{ "sourceColumn": "author_id", "referenceColumn": "id" }],
      "onDelete": "CASCADE",
      "onUpdate": "NO ACTION"
    }
  ],
  "rlsEnabled": true
}
```

`type` 欄位允許值（`columnTypeSchema`，`database.schema.ts`）包含 `string | date | datetime | integer | float | boolean | uuid | json`（或任意自訂字串型別）。

Response（`createTableResponseSchema`，`database-api.schema.ts:25-35`）：

```json
{
  "schemaName": "public",
  "tableName": "posts",
  "columns": [ /* 同上，含自動補齊的 columns */ ],
  "message": "Table 'posts' created successfully",
  "autoFields": ["id", "created_at", "updated_at"],
  "nextActions": "You can now insert records via POST /api/database/records/posts"
}
```

#### 6.8.2 `GET /api/database/admin/tables/posts/records?limit=50&offset=0`（列出記錄）

Query（`adminTableRecordsListQuerySchema`，`database-api.schema.ts:242-259`）：`limit`（1-500，預設 50）、`offset`（預設 0）、`search`、`sort`（`"col:asc,col2:desc"`）、`filterColumn`/`filterValue`（需成對出現）。

Response（`adminTableRecordsListResponseSchema`，`database-api.schema.ts:331-338`）：

```json
{
  "data": [
    { "id": "d4e5...-uuid", "title": "Hello World", "author_id": "b3f1...-uuid", "created_at": "2026-07-10T08:00:00.000Z" }
  ],
  "pagination": { "offset": 0, "limit": 50, "total": 1 }
}
```

由於使用者資料表是動態 schema，record 本身型別是 `adminTableRecordSchema = z.record(z.string(), z.unknown())`（`database-api.schema.ts:235`）——即「任意欄位鍵值對」，沒有固定的靜態型別，這與 `auth`/其他系統表使用具名 Zod schema 形成對比。

#### 6.8.3 `POST /api/database/advance/rawsql`（原生 SQL）

Request（`rawSQLRequestSchema`，`database-api.schema.ts:84-87`）：

```json
{ "query": "SELECT id, title FROM posts WHERE author_id = $1 LIMIT $2", "params": ["b3f1...-uuid", 10] }
```

Response（`rawSQLResponseSchema`，`database-api.schema.ts:89-100`）：

```json
{
  "rows": [{ "id": "d4e5...-uuid", "title": "Hello World" }],
  "rowCount": 1,
  "fields": [{ "name": "id", "dataTypeID": 2950 }, { "name": "title", "dataTypeID": 25 }]
}
```

---

## 7. `storage` domain — `/api/storage`（代表性端點）

檔案：`backend/src/api/routes/storage/index.routes.ts`（單檔，約 900 行）。

| Method | Path | 認證 | Rate limiter | 說明 |
|---|---|---|---|---|
| GET / PUT | `/config` | `verifyAdmin` | 無 | 讀取/更新 storage 設定（如 `maxFileSizeMb`） |
| GET | `/buckets` | `verifyAdmin` | 無 | 列出所有 bucket |
| POST | `/buckets` | `verifyAdmin` | 無 | 建立 bucket（`bucketName`、`isPublic`） |
| PATCH | `/buckets/:bucketName` | `verifyAdmin` | 無 | 更新 bucket 可見性 |
| DELETE | `/buckets/:bucketName` | `verifyAdmin` | 無 | 刪除整個 bucket |
| GET | `/buckets/:bucketName/objects` | `verifyUser` | 無 | 列出物件（prefix/search/分頁） |
| PUT | `/buckets/:bucketName/objects/*` | `verifyUser` | 無 | 以指定 key 上傳物件（multipart `file`） |
| POST | `/buckets/:bucketName/objects` | `verifyUser` | 無 | 上傳物件（伺服端產生 key） |
| GET | `/buckets/:bucketName/objects/*` | 條件式（公開 bucket 免驗證，否則 `verifyUser`） | 無 | 下載物件（依策略 redirect 到 presigned URL 或直接串流位元組） |
| POST | `/buckets/:bucketName/upload-strategy` | `verifyUser` | 無 | 取得上傳策略（presigned 或直接上傳） |
| POST | `/buckets/:bucketName/objects/:objectKey/confirm-upload` | `verifyUser` | 無 | 確認 presigned 上傳完成 |
| DELETE | `/buckets/:bucketName/objects` | `verifyUser` | 無 | 批次刪除物件（`keys: string[]`） |
| GET | `/s3/config` | `verifyAdmin` | 無 | S3 相容 gateway 端點與簽名 region |
| POST / GET / DELETE | `/s3/access-keys[/:id]` | `verifyAdmin` | `s3AccessKeyManagementRateLimiter` | S3 access key 建立/列出/撤銷（見 Part 2 的 `s3-gateway`） |

下載/上傳策略（presigned URL vs 直接串流）取決於 provider（S3 vs local disk），詳見 `.trace/_context/integrations.md` 第 1 節。

---

## 8. `metadata` domain — `/api/metadata`（代表性端點）

檔案：`backend/src/api/routes/metadata/index.routes.ts`。**全部路由套用 `router.use(verifyAdmin)`（全域）**，設計上是給 Dashboard／coding agent 自我內省（introspection）用的唯讀彙總 API。

| Method | Path | 說明 |
|---|---|---|
| GET | `/`（`?format=json\|markdown`） | 完整 app metadata（供 agent 一次拉取全貌，含 auth/database/storage 等所有模組摘要） |
| GET | `/auth` | Auth 模組 metadata |
| GET | `/database` | Database 模組 metadata（tables/schemas 總覽） |
| GET | `/storage` | Storage 模組 metadata（buckets 總覽） |
| GET | `/functions` | Edge functions metadata |
| GET | `/realtime` | Realtime channel metadata |
| GET | `/api-key` | 取得專案 API key |
| GET | `/anon-key` | 取得 opaque anon key |
| GET | `/project-id` | 取得 `PROJECT_ID` |
| GET | `/database-connection-string`、`/database-password` | 僅 Cloud 環境：DB 連線字串／密碼 |
| GET | `/:tableName` | 單一 table 的 schema-only metadata（catch-all，必須排在其他固定路徑之後避免遮蔽） |

`?format=markdown` 支援回傳 Markdown 格式（而非 JSON），推測是為了讓 coding agent 直接把回應塞進 prompt context（⚠️ 未驗證此用途，僅由參數命名推斷）。

---

（接續 Part 2：`API_SURFACE_part2.md`）
