# Data Flow Trace：Database Table / Record CRUD

本文件以 **Database 模組**作為代表性 use case，端對端追蹤一次請求從 dashboard 前端出發，
經過 Express 後端、驗證、business logic、persistence，最後回到 response 的完整路徑。

InsForge 的 database 模組其實有「兩條並存的資料存取路徑」，這點是整個追蹤中最關鍵的發現：

1. **Table 管理（DDL）**：`backend/src/api/routes/database/tables.routes.ts` — 純 Express + `pg` 直連
   PostgreSQL，執行 `CREATE TABLE` 等 DDL。這條路徑邏輯最完整、最適合追蹤，因此本文以
   **建立 table（`POST /api/database/tables`）**為主線。
2. **Record CRUD（DML，一般 SDK/開發者用戶）**：`backend/src/api/routes/database/records.routes.ts`
   — Express 幾乎不碰 SQL，而是把請求原封不動 **proxy 給 PostgREST** 容器。
3. **Record CRUD（Dashboard 用，admin-only）**：`backend/src/api/routes/database/admin.routes.ts`
   — 另一條 Express + 直連 `pg` 的路徑，是 dashboard UI 資料表格實際呼叫的 API（不是走 PostgREST）。

以下先完整追蹤主線（建表），再分別說明 (2)(3) 兩條 record 路徑，最後補充 frontend、error handling、
validation 章節。

---

## 主線：`POST /api/database/tables`（建立資料表）

### 1. Routing 層

檔案：`backend/src/api/routes/database/tables.routes.ts:34-76`

```
router.post('/', verifyAdmin, async (req, res, next) => { ... })
```

掛載鏈：`backend/src/api/routes/database/index.routes.ts:20` `router.use('/tables', databaseTablesRouter)`
→ 再往上掛到 `/api/database`（⚠️ 未驗證掛載點確切路徑前綴，未直接看 `backend/src/api/app.ts` 的頂層 mount，
但由 routes 目錄命名與 dashboard 呼叫路徑 `/database/tables` 可推斷前綴為 `/api`）。

### 2. Middleware 層 — `verifyAdmin`

檔案：`backend/src/api/middlewares/auth.ts:112-155`

- 先呼叫 `extractApiKey(req)`（`auth.ts:41-53`）：檢查 `Authorization: Bearer ik_...` 或
  `x-api-key` header，若命中則走 `verifyApiKey` 分支（API Key 認證，供 AI agent / CLI 使用）。
- 否則取 `Bearer` token（`extractBearerToken`, `auth.ts:32-37`），用
  `TokenManager.verifyToken(token)` 驗證 JWT。
- 要求 `payload.role === 'project_admin'`，否則丟 `AppError('Admin access required', 403,
  ERROR_CODES.AUTH_UNAUTHORIZED)`（`auth.ts:132-139`）。
- 驗證成功後 `setRequestUser(req, payload)`（`auth.ts:68-77`）把 `{id, email, role}` 掛到
  `req.user`，供後續 handler / audit log 使用。
- 任何非 `AppError` 的例外會被包成 `AppError('Invalid admin token', 401, ...)` 再 `next(error)`。

⚠️ 未驗證：本路由未見專屬的 rate limiter 或 request-body Zod middleware（validation 是在 route
handler 內手動呼叫 `safeParse`，見下），`rate-limiters.ts` 內定義的 limiter（`sendEmailOTPRateLimiter`,
`s3AccessKeyManagementRateLimiter`, `computeLogsRateLimiter`, `verifyOTPRateLimiter` 等，
`backend/src/api/middlewares/rate-limiters.ts:53-196`）皆用於 auth/S3/日誌相關端點，未套用在
database tables 路由上。

### 3. Validation 層 — Zod schema（`@insforge/shared-schemas`）

檔案：`backend/src/api/routes/database/tables.routes.ts:36-44`

```ts
const validation = createTableRequestSchema.safeParse(req.body);
if (!validation.success) {
  throw new AppError(
    validation.error.issues.map((e) => `${e.path.join('.')}: ${e.message}`).join(', '),
    400, ERROR_CODES.INVALID_INPUT, ...
  );
}
```

`createTableRequestSchema` 定義於 `packages/shared-schemas/src/database-api.schema.ts:15-23`：
是從 `tableSchema` `.pick({ tableName, columns, foreignKeys })` 再 `.extend({ rlsEnabled:
z.boolean().default(true) })`。這個 schema 同時被前端 `table.service.ts` 用作 `CreateTableRequest`
的 TypeScript type（型別與 runtime validation 共用同一份定義，前後端不會漂移）。

Zod 驗證失敗 → 直接在 route handler 內 `throw new AppError(...)`，被 catch block 的
`next(error)` 送進 error middleware（見下方「錯誤處理」章節）。

### 4. Business logic 層 — `DatabaseTableService.createTable`

檔案：`backend/src/services/database/database-table.service.ts:140-311`

流程：
1. `validateSchemaName(schemaName)` / `validateIdentifier(table_name, 'table')`
   （`backend/src/utils/validations.ts`，SQL identifier 安全性檢查，防止注入／保留字衝突）。
2. `validateReservedFields(columns)`：過濾掉與系統欄位（`id`, `created_at`, `updated_at`）同名但
   型別不符的欄位，型別不符則丟 `AppError`。
3. 若過濾後沒有任何 user-defined column，丟 `AppError(..., 400, ERROR_CODES.DATABASE_VALIDATION_ERROR)`
   （`database-table.service.ts:156-162`）。
4. 逐一 `validateIdentifier(col.columnName, 'column')` 驗證欄位名稱。
5. 從 `DatabaseManager.getInstance().getPool()` 取得 pg connection pool，`client = await
   pool.connect()`，手動開 transaction：`BEGIN` → ... → `COMMIT`／失敗則 `ROLLBACK`
   （`database-table.service.ts:181-311`）。
6. 用 `withAdminContext(client, async () => {...}, true)`
   （`backend/src/services/database/user-context.service.ts`）包裹核心邏輯 —— 這會在同一個 DB
   session 內設定 admin 身份的 session context（例如給 RLS/audit trigger 使用），確保 DDL
   以 admin 權限執行。
7. 核心 SQL 組裝（皆為手刻字串拼接 + `quoteIdentifier`／`quoteQualifiedName` 做 identifier
   quoting，而非 ORM）：
   - 檢查 table 是否已存在（查 `information_schema.tables`），存在則丟
     `AppError(..., 400, ERROR_CODES.DATABASE_DUPLICATE)`（`database-table.service.ts:203-211`）。
   - 組出每個欄位的 SQL type（透過 `COLUMN_TYPES[col.type]`，是 InsForge 自訂型別對照表，
     見 `backend/src/types/database.js`）、`DEFAULT` clause（`formatDefaultValue`，
     `database-table.service.ts:79-92`，對 `now()` / `gen_random_uuid()` 白名單放行，其餘一律
     dollar-quote 逃逸成字面值，避免 SQL injection）、`NOT NULL`、`UNIQUE`。
   - 檢查 foreign key 是否指到系統欄位（禁止），再組 FK constraint 字串。
   - 執行 `CREATE TABLE <schema>.<table> (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), <user
     columns>, created_at TIMESTAMPTZ DEFAULT now(), updated_at TIMESTAMPTZ DEFAULT now(),
     <fk constraints>); NOTIFY pgrst, 'reload schema';`（`database-table.service.ts:264-270`）。
     **`NOTIFY pgrst, 'reload schema'` 是關鍵一步**：因為 PostgREST 是另一條獨立的資料存取路徑
     （見下方章節），Express 端建表後必須主動通知 PostgREST 重新載入它快取的 schema，
     否則新表在 PostgREST REST API 上不會立刻可見。
   - 視 `use_RLS` 參數 `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`。
   - 建立 `updated_at` 自動更新 trigger（`system.update_updated_at()` function，
     ⚠️ 未驗證此 function 的定義位置，推測在某個 migration SQL 檔）。
8. `COMMIT`，回傳 `CreateTableResponse`（`schemaName`, `message`, `tableName`, `columns`
   （附上 `sqlType`）, `autoFields: ['id','created_at','updated_at']`, `nextActions` 提示字串）。

### 5. Persistence 層 — `DatabaseManager`

檔案：`backend/src/infra/database/database.manager.ts`

- `DatabaseManager` 是 singleton（`getInstance()`, line 39-44），內部持有一個 `pg.Pool`
  （`initialize()`, line 46-59，`max: 20` connections）。
- **未使用 ORM／query builder（無 Prisma/TypeORM/Knex），純 `pg` + 手寫 SQL 字串**，透過
  `pgFormat`（import at line 6）在其他地方做參數化/escaping 輔助。
- 另外維護兩個記憶體快取（非 DB 持久化）：
  - `columnTypeCache`：`getColumnTypeMap()`（line 61-94）快取每個 table 的欄位型別，TTL 5 分鐘，
    FIFO bounded（最多 100 entries，`setBoundedCache`, line 106-119）。用於
    `records.routes.ts:67` 判斷 body 內哪些欄位是「文字型別」以決定空字串是否要濾除。
  - `tableCountCache`：bounded 1000 entries, TTL 60 秒（line 31-33），用於列數統計。
  - `DatabaseManager.clearColumnTypeCache(tableName, schemaName)` 會在 `tables.routes.ts`
    建表/改表/刪表後被呼叫（line 56, 116, 148），確保快取不會過期資料。
- Migration 機制：`backend/src/infra/database/migrations/*.sql`（例如
  `004_add-reload-postgrest-func.sql`, `056_expose-custom-schemas-to-postgrest.sql`），
  ⚠️ 未驗證具體 migration runner 實作（可能是自訂 runner 而非 `node-pg-migrate`
  套件本身，檔名格式為純數字前綴 `.sql`，未見 `node-pg-migrate` 於 package.json 內逐一確認）。

### 6. Response 層

`successResponse(res, result, 201)`（`backend/src/utils/response.ts:22-24`），直接
`res.status(201).json(result)` —— InsForge 採「傳統 REST，資料直接回傳，不包一層 `{data:
...}` envelope」的風格（見檔案開頭註解 `response.ts:1-3`）。

Audit log：建表成功後（commit 之後）呼叫 `AuditService.getInstance().log({...})`
（`tables.routes.ts:58-70`），記錄 `actor`（api-key 或 `req.user.id`）、`action:
'CREATE_TABLE'`、`module: 'DATABASE'`、完整 `details`（schemaName/tableName/columns/rlsEnabled）
與 `req.ip`。這是一個「事後」的旁路寫入，不影響主交易成敗。

---

## 分支 A：Record CRUD 走 PostgREST Proxy（一般 SDK / API Key 用戶）

檔案：`backend/src/api/routes/database/records.routes.ts`

這是最終使用者 app（用 InsForge SDK）操作資料表 row 的路徑，例如
`POST /api/database/records/:tableName`。與建表路徑截然不同：**Express 完全不執行 SQL**，
只做「驗證 → 轉換 → 轉發」：

1. `router.all('/:tableName', verifyUser, forwardToPostgrest)` 與
   `router.all('/:tableName/*path', verifyUser, forwardToPostgrest)`（line 141-142）。
   `verifyUser`（`auth.ts:93-` 附近）比 `verifyAdmin` 寬鬆，接受 API key / anon key / 一般 user JWT。
2. `forwardToPostgrest`（line 34-138）：
   - `validateTableName(tableName)`（`backend/src/utils/validations.ts`）。
   - `resolvePostgrestSchema(...)`（`backend/src/services/database/helpers.js`）把 `?schema=`
     query param 轉成 PostgREST 的 `Accept-Profile` / `Content-Profile` header（PostgREST
     原生的 multi-schema 機制）。
   - 對 `POST/PATCH/PUT` body：用 `DatabaseManager.getColumnTypeMap(tableName, schemaName)`
     取欄位型別，把「非文字型別欄位的空字串值」整個過濾掉（`TEXT_LIKE_DATA_TYPES` 白名單，
     `backend/src/utils/constants.js`）—— 這是唯一的「業務邏輯」轉換，用來避免把 `''` 傳給
     `integer`/`date` 等型別造成 PostgREST/Postgres 報型別錯誤。
   - **關鍵權限轉換**（`PostgrestProxyService`，見下）：依 `req.user.role` /
     `req.hasApiKey` 決定用哪個身份的 JWT 轉發給 PostgREST：
     - `project_admin` 或帶 API key → `forwardAsAdmin`：換成內部 admin token（刻意丟棄原本
       admin 的 subject，因為 `auth.uid()` 是 UUID-based，admin subject 不是，
       `postgrest-proxy.service.ts:109-111` 註解說明）。
     - 一般已登入 user（role !== 'anon'）→ `forwardAsUser`：驗證 role 必須是
       `authenticated`/`project_admin`，用 `TokenManager.generatePostgrestUserToken({sub, email,
       role})` 產生短效 **內部 HS256 token** 轉發（`postgrest-proxy.service.ts:138-159`）。
     - 其餘（anon）→ `forwardAsAnon`：換成內部固定的 subject-less anon token
       （`postgrest-proxy.service.ts:123-131`）。
   - 實際轉發用 `axios`（`postgrestAxios`，httpAgent/httpsAgent 做 connection pooling +
     keep-alive），對 `postgrestUrl = appConfig.database.postgrestBaseUrl`（docker-compose 內為
     `http://postgrest:3000`）發請求，含最多 3 次重試（指數退避，`postgrest-proxy.service.ts:
     188-213`，只在「網路層失敗且無 response」時重試，PostgREST 回的 4xx/5xx 不重試）。
   - 回應：過濾掉不該轉發的 header（`content-length`/`transfer-encoding`/`connection`/
     `content-encoding`/`access-control-*`），空 body 轉成 `[]`，再用 `successResponse(res,
     responseData, result.status)` 直接把 PostgREST 的 status code 原樣回給 client。
   - `POST`/`DELETE` 成功後透過 `SocketManager` 廣播 `DATA_UPDATE` socket 事件給
     `role:project_admin` room，讓 dashboard 即時刷新（`records.routes.ts:120-132`）。
   - PostgREST 若回錯誤（axios error with `error.response`），`handleProxyError`
     （line 23-29）直接把 PostgREST 的 status + body 原封不動 `res.status(...).json(...)`
     回傳，**不經過 InsForge 的 `AppError`/`errorMiddleware` 正規化**；只有非 HTTP 層錯誤
     （例如連不上 PostgREST）才 `next(error)` 走 Express 標準錯誤流程。

### PostgREST 容器本身

`docker-compose.yml:26-45`：`postgrest/postgrest:v12.2.12` image，直接用
`PGRST_DB_URI` 連同一個 `postgres` container/database（`POSTGRES_DB=insforge`）。也就是說：

> **Express backend 與 PostgREST 是同一個 Postgres 資料庫的兩個獨立存取入口。**
> Express 用 `pg.Pool` 走 admin 權限直連做 DDL／管理型操作；PostgREST 是給最終應用程式用的
> 自動生成 REST API，靠 Postgres 原生角色（`anon`/`authenticated`/自訂 role）+ RLS 做權限控管，
> JWT 由 `PGRST_JWT_SECRET`（與 InsForge `JWT_SECRET` 共用同一組 secret，
> `TokenManager` 簽出的內部 token PostgREST 才驗得過）。
> `PGRST_DB_CHANNEL_ENABLED=true` + `PGRST_DB_CHANNEL=pgrst` 讓 Postgres 的
> `NOTIFY pgrst, 'reload schema'`（前面建表流程裡看到的那行）能讓 PostgREST 即時重新讀取 schema
> cache，這是兩條路徑之間唯一的「同步機制」。

---

## 分支 B：Record CRUD 走 Admin 直連（Dashboard 專用，非 PostgREST）

檔案：`backend/src/api/routes/database/admin.routes.ts`

Dashboard 前端（`packages/dashboard/src/features/database/services/record.service.ts`）
呼叫的其實是 `/database/admin/tables/:tableName/records`，**不是**
`/database/records/:tableName`（PostgREST proxy）路徑。這條路由：

- `router.use(verifyAdmin)`（`admin.routes.ts:87`）套用到整個 router，只有 project_admin
  能用（dashboard 的操作者身份）。
- `GET /tables/:tableName/records`（line 89-114）：用
  `adminTableRecordsListQuerySchema.safeParse(req.query)` 驗證分頁/搜尋/排序參數，呼叫
  `AdminRecordService`（`backend/src/services/database/admin-record.service.ts`，⚠️ 未逐行讀取
  其 SQL 實作，但依 import 與 constructor 型態判斷同樣是直連 `pg`），回傳用
  `paginatedResponse(res, records, total, offset)`（PostgREST 風格 `Content-Range` header,
  `backend/src/utils/response.ts:41-58`，故意模仿 PostgREST 的分頁慣例方便前端共用邏輯）。
- `POST`/`PATCH`/`DELETE` 對應 create/update/delete records，皆為
  「Zod schema safeParse → service 呼叫 → `broadcastRecordChange` 觸發 socket 通知 →
  `successResponse`」的固定模式，與 tables.routes.ts 手法一致。

**為何 dashboard 不直接用 PostgREST proxy？** ⚠️ 未驗證明確原因（程式碼內無註解說明），
合理推測：admin 直連可以繞過 RLS 政策看到「所有」資料（管理視角），並支援 dashboard 特有的
search/sort/filter/CSV 匯出等功能，而 PostgREST proxy 路徑是給終端應用程式用、且必須遵守
RLS。

---

## Frontend 追蹤：`packages/dashboard/src/`

### API Client

檔案：`packages/dashboard/src/lib/api/client.ts`

`ApiClient` class（非 axios，是包裝過的 `fetch`）：
- `request(endpoint, options)`（line 52-）：組 URL `${getDashboardApiBaseUrl()}${endpoint}`，
  自動注入 `Authorization: Bearer ${this.accessToken}`（若有 token，且注入順序刻意放在
  展開順序最後，註解說明是為了讓 retry 用到最新 token 而非舊的）。
- 有 `AbortSignal.timeout(REQUEST_TIMEOUT_MS)`（30 秒）與外部傳入 signal 的
  `AbortSignal.any(...)` 合併。
- 有 CSRF token 管理（`setCsrfToken`/`getCsrfToken`，走 cookie，`insforge_admin_csrf_token`）。
- 有 `onRefreshAccessToken` handler，401 時可自動 refresh 並重試一次（`skipRefresh` 旗標避免
  無限迴圈，⚠️ 未細讀 retry 判斷邏輯完整程式碼）。
- singleton 匯出 `apiClient`，供各 feature 的 `*.service.ts` 呼叫。

### Table Service + React Query hook

檔案：`packages/dashboard/src/features/database/services/table.service.ts`
與 `packages/dashboard/src/features/database/hooks/useTables.ts`

- `TableService.createTable(schemaName, tableName, columns, foreignKeys)`
  （`table.service.ts:37-52`）組出符合 `CreateTableRequest`（`@insforge/shared-schemas` 型別，
  與後端 `createTableRequestSchema` 同源）的 body，`apiClient.request('/database/tables...',
  {method: 'POST', ...})`。
- `useTables()`（`useTables.ts:12-`）用 `@tanstack/react-query`：
  - `useQuery({queryKey: databaseTableQueryKeys.tables(schemaName), queryFn: ({signal}) =>
    tableService.listTables(schemaName, signal), staleTime: 2*60*1000})` 取得表清單。
  - `createTableMutation = useMutation({mutationFn: ... tableService.createTable(...),
    onSuccess: () => queryClient.invalidateQueries(...) + showToast(success), onError: () =>
    showToast(error.message)})`：典型「mutate → invalidate query cache → toast」模式，
    沒有 optimistic update。
- Record 側對應為 `record.service.ts` + `useRecords.ts`，呼叫的是分支 B（admin 路徑）
  `/database/admin/tables/:tableName/records...`，並非 `/database/records`。

### Schema 共用

`packages/shared-schemas/src/database-api.schema.ts` 內的 Zod schema（如
`createTableRequestSchema`, `adminTableRecordsListQuerySchema` 等）同時作為：
1. 後端 route handler 的 runtime validation（`.safeParse`）。
2. 前端 TypeScript 型別來源（`z.infer<typeof ...>`），確保前後端 request/response 形狀一致，
   不需要手動同步兩份型別定義。

---

## 錯誤處理：`errorMiddleware` + `AppError`

### `AppError` 定義

檔案：`backend/src/utils/errors.ts:5-14`

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

`code` 對應 `@insforge/shared-schemas` 匯出的 `ERROR_CODES`（例如
`ERROR_CODES.INVALID_INPUT`, `DATABASE_DUPLICATE`, `AUTH_UNAUTHORIZED` 等），`nextActions`
是給呼叫端（尤其是 AI coding agent）看的「下一步該怎麼修」提示字串，很多來自
`backend/src/utils/next-actions.ts` 的 `NEXT_ACTIONS` helper（例如
`NEXT_ACTIONS.CHECK_UNIQUE_FIELD(fieldName)`）。這是 InsForge 作為「給 AI agent 用的
backend」的一個特色設計：錯誤訊息本身要對 agent 友善、可執行。

還有一個子類別 `UpstreamError extends AppError`（`errors.ts` 末段），專門包裝呼叫外部服務
（非 PostgREST proxy，PostgREST proxy 錯誤是直接透傳，見上方）失敗時的錯誤，自動從
`error.response.data`／`error.message`／`error.response.statusText` 擷取訊息與 status。

### `errorMiddleware`

檔案：`backend/src/api/middlewares/error.ts`

Express 最終錯誤處理 middleware（四參數 `(err, req, res, next)` 簽名），依序判斷：
1. `AppError` → `errorResponse(res, err.code, err.message, err.statusCode, err.nextActions)`。
   401 錯誤不記錄 `logger.error`（避免正常的未登入請求洗版 log）。
2. `SyntaxError`（JSON.parse 失敗，例如 body-parser 解析壞掉的 JSON body）→ 400 +
   `ERROR_CODES.INVALID_INPUT`。
3. `pg.DatabaseError` → `getDatabaseErrorDetails(err)`（`errors.ts` 內
   `POSTGRES_ERROR_HANDLERS` map，依 Postgres error code 轉譯，例如
   `23505` unique violation → 409 `DATABASE_DUPLICATE`，`23503` FK violation → 400
   `DATABASE_CONSTRAINT_VIOLATION`，`42501` insufficient privilege → 403 `FORBIDDEN`
   附帶 `NEXT_ACTIONS.CHECK_RLS_POLICY` 提示，等）。這代表**建表流程中若交易內任何一步
   pg 報錯，會被這裡攔截並轉成結構化 JSON**，而不是裸露的 Postgres 錯誤訊息。
4. body-parser 的 `entity.parse.failed` → 400。
5. 其餘 unknown error → 500 `ERROR_CODES.INTERNAL_ERROR`。

最終統一由 `errorResponse(res, error, message, statusCode, nextActions)`
（`backend/src/utils/response.ts:29-40`）輸出固定形狀：
`{ error, message, statusCode, nextActions }`。

### Response 慣例小結

InsForge 走「傳統 REST」：成功回應直接是 data 本身（無 envelope），錯誤回應固定包成
`{error, message, statusCode, nextActions}`（`response.ts` 檔案開頭註解明講這是刻意設計）。
分頁用 PostgREST 風格 `Content-Range` header + 206/200 status
（`paginatedResponse`, `response.ts:41-58`），而不是把 total/limit/offset 塞進 body —— 這也
是為了讓 dashboard 前端能用同一套分頁解析邏輯處理「PostgREST 原生回應」與「admin 直連回應」。

---

## 總結：兩條資料路徑的對照

| 面向 | Table DDL (`tables.routes.ts`) | Record DML／SDK 用戶 (`records.routes.ts`) | Record DML／Dashboard 用 (`admin.routes.ts`) |
|---|---|---|---|
| 認證 middleware | `verifyAdmin` | `verifyUser`（api key/anon/user JWT 皆可）| `verifyAdmin` |
| SQL 執行方 | Express，直連 `pg.Pool`，手寫 SQL | **PostgREST**（Express 只轉發）| Express，直連 `pg.Pool` |
| Validation | Zod (`createTableRequestSchema` 等) in route handler | 僅做欄位型別轉換（空字串過濾），不做 schema-level 驗證，交給 PostgREST/Postgres 自己驗 | Zod (`adminTableRecords*Schema`) in route handler |
| 權限模型 | Express 層 `role === 'project_admin'` 檢查 | Postgres RLS + role（anon/authenticated/project_admin），Express 只負責身份轉換成對應 JWT | Express 層 `role === 'project_admin'`（可能繞過 RLS）|
| 錯誤處理 | `AppError` → `errorMiddleware` → 統一 JSON | PostgREST 錯誤直接透傳原始 status/body；非 HTTP 層錯誤才走 `errorMiddleware` | `AppError` → `errorMiddleware` |
| 即時通知 | 無（僅 audit log）| Socket `DATA_UPDATE` 廣播 | Socket `DATA_UPDATE` 廣播（`broadcastRecordChange`）|

兩者共用同一個 Postgres 資料庫（`docker-compose.yml` 中 `postgres` service），差別只在
「誰負責跑 SQL、誰負責做權限判斷」。`NOTIFY pgrst, 'reload schema'`
是唯一讓兩條路徑 schema 認知保持同步的機制。

---

## 未驗證事項清單（⚠️）

- `backend/src/api/app.ts` 內 `/api/database` 的確切頂層掛載路徑與 middleware 順序未直接讀取確認。
- database migrations runner 是否為 `node-pg-migrate` 套件或自訂 runner，未確認 `package.json`
  相依與 runner 程式碼。
- `system.update_updated_at()` trigger function 的定義來源 migration 檔未定位。
- `AdminRecordService`（`admin-record.service.ts`）的 SQL 實作細節未逐行閱讀。
- dashboard 不走 PostgREST proxy 而走 admin 直連的確切設計原因（RLS bypass？功能需求？）
  程式碼內無明確註解佐證，屬推測。
- `ApiClient` 的 401 自動 refresh + retry 完整邏輯未逐行讀取。
- Auth signup/login flow（`backend/src/api/routes/auth/`）本次未追蹤，僅追蹤 database flow。
