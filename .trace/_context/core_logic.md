# InsForge 核心領域邏輯（Core Domain Logic）追蹤筆記

> 目的：定位「這個專案的心臟」——也就是讓 InsForge 有別於一般 CRUD backend
> 樣板的獨特機制。本文件為中間工作稿，聚焦具體 file:line 引用，優先完整性
> 而非文字打磨。

## 0. 全局視角

InsForge 的核心定位是「給 AI coding agent 用的 BaaS」。所有子系統
（database / auth / ai gateway / realtime）都遵循同一組重複出現的架構
模式（見第 6 節），這比任何單一子系統更能代表這個專案的「心臟」：
**用一致的 provider/manager/service 分層，把 Postgres 及第三方服務包裝
成 agent 可安全呼叫的原語（primitive）**。

真正獨特、非樣板化的邏輯集中在：

1. Dynamic table/schema management + PostgREST schema-cache 同步
   （NOTIFY/LISTEN 機制）——這是全專案最獨特的機制。
2. SQL parsing/safety layer（`libpg-query` WASM）。
3. 多 provider 的 AI Gateway（目前只有 OpenRouter 一個 concrete
   provider，但抽象層已經是 Strategy shape）。
4. Auth 的 JWT／PKCE／RLS role-switching 設計。
5. Realtime 的 pg_notify → Socket.IO 橋接（Observer / pub-sub）。

---

## 1. Dynamic Table/Schema Management + PostgREST 同步

### 1.1 建表 / 改表 / 刪表的核心

檔案：`backend/src/services/database/database-table.service.ts`

- `DatabaseTableService`（singleton，`getInstance()` 於
  `database-table.service.ts:111-116`）是 agent 用來在執行期於 Postgres
  建立/修改/刪除 table 的核心服務。
- `createTable()`（`database-table.service.ts:140-322`）：
  - 驗證 identifier（`validateIdentifier`, `validateSchemaName`，
    `database-table.service.ts:147-179`）避免 SQL injection 進 DDL。
  - 強制保留欄位 `id / created_at / updated_at`
    （`reservedColumns`，`database-table.service.ts:28-32`，
    `validateReservedFields()` `:801-821`）。
  - 用 `formatDefaultValue()` / `getSafeDollarQuotedLiteral()`
    （`:42-96`）把使用者提供的 default value 轉成安全的
    dollar-quoted literal，只白名單 `now()` 與
    `gen_random_uuid()`（`SAFE_FUNCS`，`:34`）當作可執行函式，其餘一律
    當作字串常數逸出，避免函式呼叫注入。
  - 在同一個 transaction 中：`CREATE TABLE` → 視需要
    `ENABLE ROW LEVEL SECURITY`（`:273-280`，預設 `use_RLS = true`）→
    建立 `updated_at` 的 `BEFORE UPDATE` trigger
    （`system.update_updated_at()`，`:283-289`）→ **`NOTIFY pgrst,
    'reload schema';`**（`:266-271`，緊跟在 `CREATE TABLE` 語句後）。
  - 全程包在 `withAdminContext(client, fn, true)`
    （`database-table.service.ts:187-308`，定義見 1.2）以
    `project_admin` role 執行，繞過一般使用者的 RLS 限制。
- `updateTableSchema()`（`:464-736`）支援 add/drop/update column、
  add/drop foreign key、rename table，每個分支都拒絕碰保留欄位
  （例：`:563-570` drop system column 會擲出
  `DATABASE_FORBIDDEN`），完成後同樣呼叫
  `NOTIFY pgrst, 'reload schema';`（`:708-712`）。
- `deleteTable()`（`:741-789`）用 `DROP TABLE IF EXISTS ... CASCADE`
  ＋ 同一個 NOTIFY（`:760-764`）。

### 1.2 RLS Role-Switching：`withAdminContext` / `withUserContext`

檔案：`backend/src/services/database/user-context.service.ts`

- `withUserContext()`（`:25-80`）：一般使用者/匿名請求的路徑。在單一
  transaction 內 `SET LOCAL ROLE authenticated|anon|project_admin`
  （因為 Postgres 的 bind parameter 不能綁 identifier，所以用白名單式
  字串插值，註解明確警告未來不要把它改成從 JSON payload 動態建構
  ——`:45-49`），再用 `set_config('request.jwt.claims', ...)`
  把 JWT claims 寫進 session-local 設定，讓 RLS policy 內的
  `auth.uid()` 之類的 helper function 可讀到。`finally` 一定
  `RESET ROLE`（`:76-79`）避免 role 洩漏到連線池的下一個使用者。
- `withAdminContext()`（`:90-149`）：DDL 操作（建表/改表/刪表/raw SQL
  as-admin）用的版本，`SET (LOCAL) ROLE project_admin`，並用
  `set_config` 寫入 `{role:'project_admin'}` claims。錯誤處理刻意做了
  `pendingError` / `cleanupError` 的 `cause` chain（`:134-146`）確保
  cleanup 失敗不會吃掉原始錯誤。

⚠️ 未驗證：`system.update_updated_at()` trigger function 本身定義（在
migration SQL 中，未直接讀取原始碼確認內容，只確認被呼叫）。

### 1.3 NOTIFY pgrst 機制（PostgREST schema cache 同步）— 全專案最獨特的機制

`grep -rn "NOTIFY\|pgrst" backend/src` 命中以下所有位置，代表凡是會改變
資料庫 schema 的路徑都要手動觸發 PostgREST reload：

- `database-table.service.ts:269,710,762`（createTable / updateTableSchema
  / deleteTable）
- `database-advance.service.ts:110,821,1023`（raw SQL 執行後偵測
  `/CREATE|ALTER|DROP/i` 就補發 NOTIFY，見 `:108-112`；bulk
  import/upsert 相關路徑）
- `database-backup.service.ts:307`（DB restore 後）
- `database-migration.service.ts:114`（migration apply 後）

機制本質：PostgREST 會快取一份 DB schema（用來產生 REST 路由），
InsForge 自己執行 DDL 不會讓 PostgREST 自動感知，所以每次 DDL commit
後都用 Postgres 原生的 `LISTEN/NOTIFY` pub-sub 傳送 `pgrst` channel 上的
`'reload schema'` 訊息（PostgREST 內建監聽此 channel 名稱的慣例）。
這讓「agent 呼叫 API 建表」→「同一個 project 的 REST API
立刻能查詢新表」形成閉環，且不需要重啟 PostgREST 或輪詢 schema。

`backend/src/infra/database/database.manager.ts:246` 附近的註解也提到
「Create a dedicated client for operations that can't use pooled
connections (e.g., LISTEN/NOTIFY)」——這與 realtime 訂閱用的
`DatabaseManager.getInstance().createClient()`（見第 5 節）是同一套
基礎設施，NOTIFY/LISTEN 是 InsForge 內部貫穿 schema-sync 與
realtime pub/sub 兩個子系統的共用機制。

### 1.4 SQL Parser 在 dynamic schema 場景的角色

`database-advance.service.ts` 中的 `executeRawSQL()`
（`:83-137`）是「raw SQL passthrough」端點的核心：呼叫
`sanitizeQuery()`（`:71-81`，內部呼叫 `checkSqlExecutionGuards`，見
第 2 節）過濾危險語句，再依 `asRoot` 參數決定要不要包進
`withAdminContext`，執行後用 regex `/CREATE|ALTER|DROP/i`
（`:109`）粗略判斷是否為 DDL 並補發 `NOTIFY pgrst`。

---

## 2. SQL Parsing / Safety — `backend/src/utils/sql-parser.ts`

底層使用 `libpg-query`（Postgres 官方 parser 移植成 WASM，`parseSync` /
`loadModule` from `libpg-query`，`sql-parser.ts:3`）。`initSqlParser()`
（`:39-46`）在 `backend/src/server.ts:74` 啟動時 await 一次，把 WASM
模組載入常駐記憶體，避免每次請求都重新初始化。

三個主要輸出，服務三種不同用途：

1. **`analyzeQuery(query)`**（`:86-127`）：把 SQL AST 轉成
   `DatabaseResourceUpdate[]`（`type: tables|table|records|index|
   trigger|policy|function|extension|migration`），用 `STMT_TYPES`
   （`:62-75`）和 `DROP_TYPES`（`:77-84`）兩個 lookup table 把
   libpg-query 回傳的 statement node 型別（`CreateStmt` /
   `AlterTableStmt` / `InsertStmt`…）映射成語意化的「這條 SQL 動了什麼
   資源」。用途：讓上層（例如 audit log / metadata 刷新判斷）知道一段
   raw SQL 實際影響哪些資源類型，而不用重寫正規表示式硬猜。

2. **`checkSqlExecutionGuards(query)`**（`:129-177`）：**注入/越權防護
   的核心關卡**。解析每個 statement，明確擋下：
   - `DATABASE_MANAGEMENT_STATEMENTS`（`CreatedbStmt` /
     `DropdbStmt` / `AlterDatabaseStmt` 等，`:19-25`）——不准動資料庫層級
     物件。
   - `VariableSetStmt` 中對 `role` / `session_authorization`
     （`EXECUTION_CONTEXT_VARIABLES`，`:8`）、`search_path`
     （`:17`）、`statement_timeout`（`:9`）的變更——防止 agent 用
     `SET ROLE` 提權或改變安全邊界（`:146-161`）。
   - `RESET ALL`（`:147-149`）。
   - `ROLE_MANAGEMENT_STATEMENTS`（`CreateRoleStmt` /
     `AlterRoleStmt` / `DropRoleStmt` / `GrantRoleStmt`，
     `:10-16`）——完全禁止角色管理。
   - `TransactionStmt`（`BEGIN/COMMIT/ROLLBACK` 等，`:167-169`）——不准
     使用者的 raw SQL 自己控制 transaction 邊界（外層服務已經用自己的
     transaction 包住）。
   - 額外用純文字 regex `SET_CONFIG_PATTERN = /\bset_config\b/i`
     （`:18`, `:27-33`）擋 `set_config()` 函式呼叫——這條在 AST
     guard 之前先跑（`:130-133`），因為 `set_config` 可能藏在
     `SELECT` 語句裡而不是 `VariableSetStmt`，用純 AST 型別判斷會漏。
   - 解析失敗（`parseSync` throw）直接判定拒絕（`:172-176`,
     fail-closed 設計：parse 不出來就不執行）。

3. **`parseSQLStatements(sqlText)`**（`:190-226`）：用
   `@databases/split-sql-query` + `@databases/sql` 把一段可能包含多條
   語句、字串常數內嵌分號、註解的 SQL 文字安全切成多個獨立語句陣列，
   讓上層可以逐條執行 / 逐條套用 guard，而不會被字串裡的 `;` 誤切。

⚠️ 未驗證：`checkSqlExecutionGuards` 是否覆蓋了所有可能的提權路徑
（例如透過 extension function 間接改變 session 狀態），僅描述目前程式
碼中列出的白名單/黑名單。

---

## 3. AI / Model Gateway

### 3.1 分層

- `backend/src/services/ai/chat-completion.service.ts`：`ChatCompletionService`
  singleton（`:62-73`），對外提供 OpenAI-compatible 的 `chat()` /
  `streamChat()`。內部把 InsForge 自家的 `ChatMessageSchema` 轉成
  `OpenAI.Chat.ChatCompletionMessageParam`（`formatMessages()`,
  `:78-143`），支援 tool_calls、多模態 image、annotations（web
  citation）等 OpenAI 語意的擴充；並在 `buildPlugins()`
  （`:202-216`）注入 OpenRouter 專屬的 `web` 搜尋和 `file-parser`
  （PDF）plugin。
- `backend/src/providers/ai/openrouter.provider.ts`：`OpenRouterProvider`
  singleton（`:84-99`），是真正打 HTTP 出去的層。核心是
  `sendRequest<T>(request: (client: OpenAI) => Promise<T>)`
  （`:677-730`）——一個高階函式包裝，呼叫端傳入「拿到 client 後要做
  什麼」的 callback，provider 負責解析 API key、建立/快取
  `OpenAI` SDK client、執行、並把 OpenRouter 特有的錯誤（402 額度用盡、
  401/403 金鑰錯、429 rate limit）轉譯成 InsForge 自家的
  `AppError` + `ERROR_CODES`（`:688-722`）。

### 3.2 API Key 來源策略（Strategy-ish）

`getApiKeyWithSource()`（`:121-139`）依環境二選一：
- Cloud 環境（`isCloudEnvironment()`）→ `fetchCloudApiKey()`
  （`:550-606`）向 `CLOUD_API_HOST`（`api.insforge.dev`）用簽名 JWT
  （`createCloudProjectToken()`, `:479-489`）換取 InsForge 代管的
  OpenRouter key，並用 `fetchPromise` 做 promise memoization
  （`:558-562`）避免併發重複請求。
- 自架（self-hosted）→ 直接讀環境變數 `OPENROUTER_API_KEY`
  （`:130-138`）。

`rotateCloudApiKey()`（`:608-667`）處理金鑰輪替：用
`rotationPromise` 鎖住併發呼叫，並且刻意等待任何進行中的
`fetchPromise` 先完成（`:619-622`，註解解釋：避免 fetch 的回應在
rotate 之後才落地，把 cloudCredentials 又蓋回舊金鑰）。

### 3.3 Provider 抽象是否為完整 Strategy Pattern？

目前 `backend/src/providers/ai/` 底下只有一個 concrete provider
（OpenRouter），**沒有共用 interface**（不像 `providers/oauth/` 有
`OAuthProvider` interface，見第 6 節）。所以 AI Gateway 目前更像是
「single-provider gateway + provider-internal 的 cloud/env 兩種
key-source 策略」，而不是可插拔的多 LLM provider abstraction。
OpenRouter 本身是一個聚合多家 LLM（OpenAI/Anthropic/Google 等）的
代理服務，所以 InsForge 是把「支援多模型」這件事外包給 OpenRouter 的
`model` 字串（例如 `buildModelId()`, `chat-completion.service.ts:148-153`
處理 `:thinking` 後綴），而不是自己在 backend 層實作多個
provider adapter。

⚠️ 未驗證：`backend/src/services/ai/embedding.service.ts` 與
`image-generation.service.ts` 是否也都只走 OpenRouter，未逐一讀取確認
（但目錄下沒有其他 `providers/ai/*.provider.ts`，可合理推斷是同一條
路徑）。

---

## 4. Auth 核心

### 4.1 JWT 簽發與雙演算法策略

檔案：`backend/src/infra/security/token.manager.ts`

- `TokenManager`（singleton，`:46-65`）優先用 RS256
  非對稱金鑰簽 access token（`generateAccessToken()`,
  `:145-157`），金鑰透過 `ensureKeysLoaded()`（`:70-111`）從
  `SecretService` 讀取（DB-backed secret store），若尚不存在則呼叫
  `secretService.initializeJwtKeyPair()` 首次產生；若金鑰不可用則
  fallback 回 HS256 + `JWT_SECRET` 環境變數。
  `verifyToken()`（`:272-310`）對稱地：先看 JWT header 的
  `kid`/`alg` 是否匹配已載入的 RS256 公鑰，否則 fallback 驗證 HS256。
- 另外維護三種「用途限定」的 token：
  - `generatePostgrestUserToken()`（`:163-168`）：5 分鐘短效期，僅供
    後端把已驗證使用者的請求轉發給 PostgREST 時使用。
  - `generatePostgrestAdminToken()` / `generatePostgrestAnonToken()`
    （`:174-182`, `:259-267`）：**永不過期**、無 subject 的內部 token，
    分別代表 `project_admin` 和 `anon` 角色（`:246-258` 註解特別強調
    anon token 沒有 subject，所以 `auth.uid()` 會是 NULL，避免和真人
    使用者的身分綁定）。這兩個 token 只在 server 內部使用，
    不會流出到 client（見 `postgrest-proxy.service.ts:64-114` 的
    `forwardAsAdmin()`）。
- Refresh token 設計（`generateRefreshToken()` /
  `generateRefreshTokenWithCsrf()`, `:187-212`）：7 天效期，帶
  `sessionType: 'user'|'admin'` 與隨機 `csrfNonce`；`verifyCsrfToken()`
  （`:370-381`）用 HMAC-SHA256 重新計算並 `crypto.timingSafeEqual`
  比較，防止 timing attack 洩漏 CSRF token。
- 也支援驗證 InsForge Cloud 簽發的 JWT
  （`verifyCloudToken()`, `:316-354`，用 `createRemoteJWKSet` 對
  `${cloudApiHost}/.well-known/jwks.json` 做 JWKS 快取驗簽）。

### 4.2 OAuth PKCE Flow

檔案：`backend/src/services/auth/oauth-pkce.service.ts`

- `OAuthPKCEService`（singleton，`:26-47`）目的是讓 OAuth callback URL
  不直接帶 access token（避免 token 出現在瀏覽器歷史/referrer），而是
  簽發一次性、短效期（`CODE_EXPIRY_MINUTES = 5`, `:33`）的
  exchange code。
- `createCode()`（`:61-78`）：OAuth provider callback 成功後，用
  `crypto.randomBytes` 產生 32 bytes 隨機 code（`generateSecureToken`），
  存進記憶體內的 `Map<string, PKCECodeData>`（`:29`）——**注意這是
  process-local in-memory store，非資料庫**，代表多實例部署時 PKCE
  code 不能跨實例交換（⚠️ 未驗證是否有多實例場景的相應設計）。
- `exchangeCode()`（`:83-127`）：查 code → 立即刪除（one-time use，
  `:91`）→ 檢查過期 → 用 SHA256(code_verifier) 與存好的
  `codeChallenge` 比對（PKCE 核心驗證，`:100-104`）→ 通過後才呼叫
  `TokenManager.generateAccessToken()` 現場鑄造 access token
  （`:115-119`）。
- `cleanupExpiredCodes()`（`:132-146`）由 constructor 裡的
  `setInterval`（每 5 分鐘，`:37-38`）驅動，是一個典型的
  in-memory TTL cache 自清理模式。

### 4.3 RLS Role-Switching 與 Auth 的關聯

`auth.service.ts` 的 `AuthService`（singleton，`:55-96`）在 constructor
就把八個 OAuth provider（Google/GitHub/Discord/LinkedIn/Facebook/
Microsoft/X/Apple，`:62-93`）全部 instantiate 成快取好的 singleton
成員變數——這是 6 個以上 OAuth vendor 共用同一個 `OAuthProvider`
interface 的 Strategy pattern 具體案例（interface 定義見
`backend/src/providers/oauth/base.provider.ts:7-29`：
`generateOAuthUrl()` / `handleCallback()` / 可選的
`handleSharedCallback()`）。

最終使用者的請求都要通過 `withUserContext()`
（見 1.2）把 JWT payload 轉成 Postgres session 的 `role` 與
`request.jwt.claims`，讓 RLS policy 生效——這把「JWT 驗證」和「資料庫
列級權限」串成同一條 pipeline。

---

## 5. Realtime：pg_notify → Socket.IO 橋接

### 5.1 pg_notify 監聽層

檔案：`backend/src/infra/realtime/realtime.manager.ts`

- `RealtimeManager`（singleton，`:21-40`）在
  `initialize()`（`:45-80`）用
  `DatabaseManager.getInstance().createClient()`
  （**不走連線池**，因為 `LISTEN` 需要專屬長連線 —— 註解見
  `database.manager.ts:246`）建立一條專屬連線，執行
  `LISTEN realtime_message`（`:55`），並掛上 `notification` /
  `error` / `end` 三個事件 handler（`:59-73`）。
- payload 設計刻意精簡：`handlePGNotification(messageId)`
  （`:86-120`）只收 message 的 UUID（註解明講「bypass 8KB limit」，
  即 Postgres NOTIFY payload 上限），實際訊息內容再用
  `RealtimeMessageService.getInstance().getById(messageId)`
  （`:90`）回查資料庫——這是「輕量通知 + 拉取完整資料」的常見
  pub/sub 優化模式。
- `publishMessage()`（`:125-152`）雙路徑扇出：
  - WebSocket：`publishToWebSocket()`（`:158-176`）透過
    `SocketManager.getInstance().broadcastToRoom()` 把訊息丟進
    `realtime:${channelName}` 這個 Socket.IO room。
  - Webhook：`publishToWebhooks()`（`:181-190`）用
    `WebhookSender`（`backend/src/infra/realtime/webhook-sender.ts`）
    做 HTTP POST 扇出到 channel 設定的 URL 清單。
  - 兩者結果都會回寫進 `RealtimeMessageService.updateDeliveryStats()`
    （`:109`），做送達統計。
- 斷線重連用指數退避（`handleDisconnect()`, `:195-220`：
  `baseReconnectDelay=5000ms`、`maxReconnectAttempts=10`，
  delay = `baseReconnectDelay * 2^attempts`）。

### 5.2 Socket.IO 層

檔案：`backend/src/infra/socket/socket.manager.ts`

- `SocketManager`（singleton，`:31-46`，並在檔案末尾額外 export
  一個模組級 `socketService` 常數，`:664`，方便其他模組直接
  import 用）。
- 三種認證管道在同一個 `io.use()` middleware
  （`setupMiddleware()`, `:68-167`）里按優先序判斷：
  1. API key（`apiKey` handshake 欄位，`:80-98`）→ 角色
     `project_admin`。
  2. Opaque anon key（`token` 以 `anon_` 開頭，`:104-123`）→ 角色
     `anon`，sentinel subject `'anonymous'`。
  3. 一般 JWT（`:125-165`，呼叫 `tokenManager.verifyToken()`）→
     使用者角色。
  這三種認證方式與 REST/PostgREST 那邊的三種角色（`authenticated` /
  `anon` / `project_admin`）完全對應，顯示 InsForge 把「連線層身份」
  與「資料庫 RLS 角色」設計成同一套語彙。
- 訂閱／發佈：`handleRealtimeSubscribe()`（`:302-377`）會先呼叫
  `RealtimeAuthService.checkSubscribePermission()`
  （透過 RLS 的 SELECT policy 判斷，函式命名直接點名
  「Check subscribe permission via RLS SELECT policy」，`:312`）
  才准 `socket.join(roomName)`；`handleRealtimePublish()`
  （`:411-458`）則是直接把訊息 `insertMessage()`
  進資料庫，並非直接 emit——**由資料庫 trigger 負責觸發
  pg_notify，再由 5.1 的 RealtimeManager 監聽並廣播**（註解
  `:409`：「Inserts message to DB - trigger handles pg_notify,
  broadcast, and stats update」）。也就是說：即使是
  client 端「即時」發出的訊息，也一律先落地到 Postgres，再走
  `NOTIFY realtime_message` → `RealtimeManager` → `SocketManager`
  這條唯一路徑廣播，而不是繞過資料庫直接 socket-to-socket 廣播。
  這保證了資料一致性（有落庫記錄）與單一事實來源。
- Presence（線上狀態）透過 `RealtimePresenceService`
  （`realtime-presence.service.ts`，`trackMember()` /
  `removeSocketFromRoom()` / `removeSocketFromAllRooms()`）在
  `onSocketDisconnect()`（`:228-252`）與訂閱/取消訂閱流程中維護，
  並廣播 `PRESENCE_JOIN` / `PRESENCE_LEAVE` 事件
  （`emitPresenceMemberEvent()`, `:619-638`）。

### 5.3 Observer / Pub-Sub 總結

整條鏈路是教科書式的 Observer/pub-sub：
`Postgres row insert (trigger)` → `pg_notify('realtime_message',
messageId)` → `RealtimeManager`（唯一 subscriber，用專屬連線
`LISTEN`）→ 依 channel 設定 fan-out 到 `SocketManager`（WebSocket
room broadcast）與 `WebhookSender`（HTTP POST 多個 URL）。
`DatabaseManager.createClient()` 建立「非池化」的專屬連線，是這整套
機制能運作的基礎設施前提（1.3 節提到的 schema-reload NOTIFY 也依賴
同一種專屬連線能力）。

---

## 6. 貫穿全專案的設計模式

### 6.1 Singleton（`getInstance()`）—— 幾乎每個 service/manager 都用

**每一個** 被追蹤到的核心類別都是私有 constructor + 靜態
`getInstance()` 的 singleton：
- `DatabaseTableService.getInstance()`
  （`database-table.service.ts:111-116`）
- `DatabaseAdvanceService.getInstance()`
  （`database-advance.service.ts:27-32`）
- `MetadataService.getInstance()`（`metadata.service.ts:19-24`）
- `TokenManager.getInstance()`（`token.manager.ts:60-65`）
- `AuthService.getInstance()`（`auth.service.ts:98-102`）
- `OAuthPKCEService.getInstance()`
  （`oauth-pkce.service.ts:42-47`）
- `OpenRouterProvider.getInstance()`
  （`openrouter.provider.ts:94-99`）
- `ChatCompletionService.getInstance()`
  （`chat-completion.service.ts:68-73`）
- `RealtimeManager.getInstance()`
  （`realtime.manager.ts:35-40`）
- `SocketManager.getInstance()`（`socket.manager.ts:41-46`，並額外
  export 一個 module-level 便利常數 `socketService`）
- `PostgrestProxyService.getInstance()`
  （`postgrest-proxy.service.ts:75-80`）

這個模式服務兩個目的：(a) 昂貴資源（DB pool、Socket.IO server、
OAuth provider client、long-lived LISTEN 連線）只建立一次；
(b) 讓 Express route handler 可以在任何地方直接
`XxxService.getInstance()` 取用，不需要依賴注入容器。

### 6.2 Strategy Pattern —— `providers/` 目錄下的多實作 + 共用 interface

`backend/src/providers/` 底下每個子目錄都是同一形狀：一個
`base.provider.ts` 定義 interface/abstract class，多個
concrete `*.provider.ts` 實作它，由上層 service 在 runtime 依設定
選用。具體例證：

- **OAuth**：`providers/oauth/base.provider.ts:7-29` 定義
  `OAuthProvider` interface（`generateOAuthUrl` /
  `handleCallback` / 可選 `handleSharedCallback`），8 個 provider
  （google/github/discord/linkedin/facebook/microsoft/x/apple/
  custom）各自實作，`AuthService` 在 constructor 一次性
  instantiate 全部（`auth.service.ts:86-93`）。
- **Storage**：`providers/storage/base.provider.ts` +
  `s3.provider.ts` / `local.provider.ts`（依 S3 相容或本機檔案系統
  切換）。
- **Database provisioning**：`providers/database/base.provider.ts`
  + `cloud.provider.ts`（cloud-managed Postgres provisioning）。
- **Email**：`providers/email/base.provider.ts` + `smtp.provider.ts`
  / `cloud.provider.ts`。
- **Logs**：`providers/logs/base.provider.ts` +
  `cloudwatch.provider.ts` / `local.provider.ts`。
- **Compute**：`providers/compute/compute.provider.ts` +
  `fly.provider.ts` / `cloud.provider.ts`。
- **Payments**：`providers/payments/stripe.provider.ts` /
  `razorpay.provider.ts`（+ 各自的 error-mapping 檔）。
- **AI**：`providers/ai/openrouter.provider.ts` 目前只有單一實作，
  沒有共用 interface（見 3.3 節）——是這個模式中唯一的例外，
  值得注意 InsForge 尚未把「多 LLM vendor」抽象化到 backend 層，而是
  委託給 OpenRouter 本身聚合多模型。

這組 `base.provider.ts` + N 個 `*.provider.ts` 的重複結構，是
InsForge 用來讓「self-hosted vs. cloud」兩種部署形態共用同一套
service 邏輯的關鍵設計：service 層只依賴 interface，
provider 由環境（`isCloudEnvironment()`）或設定決定實際注入哪個
實作。

### 6.3 Pipeline —— transaction 內的多步驟 DDL/DML 操作

`createTable()` / `updateTableSchema()` / `executeRawSQL()`
（database-table.service.ts, database-advance.service.ts）都遵循
同一個 pipeline 形狀：
`BEGIN → validate → withAdminContext(SET ROLE) → 執行 SQL →
NOTIFY pgrst → COMMIT`（失敗則 `ROLLBACK`）。這個固定順序在三個檔案
中重複出現，可視為一種手寫的（非框架化的）pipeline pattern。

### 6.4 Observer / Pub-Sub —— Realtime 子系統

見第 5 節。`pg_notify` 是 publisher，`RealtimeManager` 是唯一
subscriber，`SocketManager` 房間廣播與 `WebhookSender` 是兩個
observer/fan-out 端點。Schema-reload 機制（1.3 節）用同一種
`NOTIFY/LISTEN` 原語，但 subscriber 換成 PostgREST 自己（外部進程），
InsForge backend 只負責 publish 端。

### 6.5 Promise Memoization —— 防止並發重複外部呼叫

在 `OpenRouterProvider`（`fetchPromise` / `rotationPromise`，
`openrouter.provider.ts:89-90, 558-606, 608-667`）與
`TokenManager`（`loadPromise`，`token.manager.ts:52, 74-111`）中
重複出現同一個小 pattern：把「進行中的非同步呼叫」快取成一個共享
Promise，讓並發呼叫者 await 同一個 in-flight 請求，而不是各自重複
打外部 API 或重複初始化金鑰。

---

## 7. 待進一步驗證的問題（給後續 trace 用）

- ⚠️ `system.update_updated_at()` trigger function 的實際 SQL 定義
  未讀取（推測在某個 migration 檔案或 bootstrap SQL 中）。
- ⚠️ `database-migration.service.ts` 完整內容未讀（只確認其中一處
  `NOTIFY pgrst`），值得追查它與 `database-table.service.ts` 的
  DDL 邏輯是否有重複/分工。
- ⚠️ OAuthPKCEService 用 in-memory `Map` 存 PKCE code，若後端跑
  多實例（例如 cloud 環境水平擴展），code 只在簽發它的那個實例
  記憶體裡，換一台實例接手 callback 可能會失敗——未驗證是否有
  sticky routing 或改用 Redis/DB 的計畫。
- ⚠️ AI Gateway 除了 `chat-completion.service.ts`，還有
  `embedding.service.ts` / `image-generation.service.ts`，未逐一確認
  是否也走 `OpenRouterProvider.sendRequest()` 同一條路徑。
