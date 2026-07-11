# InsForge API 介面參考文件（Part 2／2）

> 承接 `API_SURFACE_part1.md`。本檔涵蓋除 `auth`/`database`/`storage`/`metadata` 外，其餘 18 個 route domain：
> `logs`／`docs`／`functions`／`secrets`／`usage`／`ai`／`memory`／`realtime`／`email`／`deployments`／`schedules`／`payments`／`compute`／`analytics`／`webscraper`／`webhooks`／`s3-gateway`／`advisor`。
>
> 除 `webhooks`／`s3-gateway` 外，其餘皆掛載在 `/api/<domain>`。認證 middleware 定義見 Part 1 第 1 節。Rate limiter 定義於 `backend/src/api/middlewares/rate-limiters.ts`。

---

## 9. `logs` domain — `/api/logs`

檔案：`backend/src/api/routes/logs/index.routes.ts`。**`router.use(verifyAdmin)` 全域套用**，無專屬 rate limiter。

| Method | Path | 說明 |
|---|---|---|
| GET | `/audits` | 列出審計日誌記錄（分頁） |
| GET | `/audits/stats` | 審計日誌統計（`days` 區間） |
| DELETE | `/audits` | 清除超過 `days_to_keep` 的審計日誌 |
| GET | `/sources` | 列出日誌來源（insforge/postgrest/postgres/function 等元件） |
| GET | `/stats` | 跨所有來源的統計 |
| GET | `/functions/build-logs` | Deno Deploy function 建置日誌 |
| GET | `/search` | 跨日誌全文搜尋 |
| GET | `/:source` | 特定來源的日誌 |

用途：admin-only 系統／審計日誌查詢與搜尋，後端由 CloudWatch 或本地檔案 provider 支援（見 `.trace/_context/integrations.md` 第 8 節）。

---

## 10. `docs` domain — `/api/docs`

檔案：`backend/src/api/routes/docs/index.routes.ts`。**無任何認證 middleware**（公開路由，供 Dashboard／agent 讀取文件內容）。

| Method | Path | 說明 |
|---|---|---|
| GET | `/:docType` | 舊版文件查詢（如 `db-sdk`、`auth-sdk`、`real-time`、`payments`） |
| GET | `/:docFeature/:docLanguage` | 依「功能 × 語言」查 SDK 文件（typescript/swift/kotlin/rest-api） |
| GET | `/` | 列出所有可用文件條目/端點 |

用途：伺服 MDX 文件內容（含 snippet-import 解析與路徑穿越防護），供 Dashboard 或 coding agent 內建文件瀏覽器使用。

---

## 11. `functions` domain — `/api/functions`

檔案：`backend/src/api/routes/functions/index.routes.ts`。認證：`verifyAdmin`。Rate limiter：`functionsWriteLimiter`（寫入操作）。

| Method | Path | 認證 | Rate limiter | 說明 |
|---|---|---|---|---|
| GET | `/` | `verifyAdmin` | 無 | 列出 edge functions |
| GET | `/:slug` | `verifyAdmin` | 無 | 取得函式詳情＋原始碼 |
| POST | `/` | `verifyAdmin` | `functionsWriteLimiter` | 建立函式（部署到 Deno Deploy），廣播 socket 事件 |
| PUT | `/:slug` | `verifyAdmin` | `functionsWriteLimiter` | 更新函式 |
| DELETE | `/:slug` | `verifyAdmin` | `functionsWriteLimiter` | 刪除函式 |

用途：edge function 的 CRUD 與部署管理，操作皆有審計記錄。實際函式**執行**走另一條相容路徑：`app.all('/functions/:slug', ...)`（`server.ts:231-289`，不在 `/api` prefix 下），轉發到 Deno Deploy 或本地 Deno runtime（見 `.trace/_context/entry_points.md` 1.2 節第 15 點）。

---

## 12. `secrets` domain — `/api/secrets`

檔案：`backend/src/api/routes/secrets/index.routes.ts`。**全部路由 `verifyAdmin`**，無專屬 rate limiter。

| Method | Path | 說明 |
|---|---|---|
| GET | `/` | 列出 secrets（僅 metadata，不含明文值） |
| GET | `/:key` | 取得解密後的 secret 值 |
| POST | `/` | 建立 secret（key 需符合 `^[A-Z0-9_]+$`） |
| PUT | `/:key` | 更新 secret（`isReserved` 的保留 key 禁止更新） |
| POST | `/api-key/rotate` | 輪替專案 API key（有寬限期） |
| POST | `/anon-key/rotate` | 輪替 anon key（有寬限期） |
| DELETE | `/:key` | 軟刪除（標記 inactive），保留 key 禁止刪除 |

用途：管理加密的專案 secrets／環境變數，供 edge functions 使用；secret 變更會觸發 debounced 函式重新部署。

---

## 13. `usage` domain — `/api/usage`

檔案：`backend/src/api/routes/usage/index.routes.ts`。認證因端點而異。

| Method | Path | 認證 | 說明 |
|---|---|---|---|
| POST | `/mcp` | `verifyApiKey` | 記錄 MCP 工具使用事件，廣播 `MCP_CONNECTED` socket 事件 |
| GET | `/mcp` | `verifyAdmin` | 列出 MCP 使用記錄 |
| GET | `/stats` | `verifyCloudBackend` | 指定日期區間的使用統計（供 InsForge Cloud 後端呼叫） |

用途：MCP 工具遙測與使用量統計，`/stats` 是唯一使用 `verifyCloudBackend` 的 domain 範例（backend-to-backend，非一般客戶端可呼叫）。

---

## 14. `ai` domain — `/api/ai`（AI Model Gateway，OpenRouter-backed）

檔案：`backend/src/api/routes/ai/index.routes.ts`。管理端 `verifyAdmin`，推論端 `verifyUser`。無專屬 rate limiter（依賴上游 OpenRouter 的 429）。

| Method | Path | 認證 | 說明 |
|---|---|---|---|
| GET | `/models` | `verifyAdmin` | 列出可用模型 |
| GET | `/overview` | `verifyAdmin` | OpenRouter key 層級的 Gateway 可觀測性資訊 |
| GET | `/:provider/api-key` | `verifyAdmin` | 取得遮罩後的 provider API key |
| POST | `/:provider/api-key/rotate` | `verifyAdmin` | 輪替代管 provider key |
| POST | `/chat/completion` | `verifyUser` | Chat completion（支援 SSE streaming） |
| POST | `/image/generation` | `verifyUser` | 圖片生成 |
| POST | `/embeddings` | `verifyUser` | 文字向量化 |

失敗處理策略見 `.trace/_context/integrations.md` 第 5 節：僅偵測並回報 429，不自動 retry。

---

## 15. `memory` domain — `/api/memory`

檔案：`backend/src/api/routes/memory/index.routes.ts`。**`router.use(verifyApiKey)` 全域套用**——是唯一整體僅接受 API key（不接受一般 JWT）的 domain，定位為「平台管理的、CLI/agent 專用」原語。

| Method | Path | 說明 |
|---|---|---|
| POST | `/remember` | 儲存記憶條目 |
| POST | `/recall` | 檢索／搜尋記憶 |
| POST | `/index` | 便宜的「僅標題」列表（always-load tier），依 `scope` 篩選 |

用途：agent 長期記憶儲存與檢索。

---

## 16. `realtime` domain — `/api/realtime`

檔案：4 個檔案（`index.routes.ts` + `channels.routes.ts` + `messages.routes.ts` + `permissions.routes.ts`）。**全部 `verifyAdmin`**，無專屬 rate limiter。

| Method | Path | 說明 |
|---|---|---|
| GET / PATCH | `/config` | 讀取／更新訊息保留設定 |
| GET / POST | `/channels` | 列出／建立 channel |
| GET / PUT / DELETE | `/channels/:id` | 單一 channel CRUD |
| GET | `/messages` | 列出訊息歷史 |
| DELETE | `/messages` | 清空所有訊息 |
| GET | `/messages/stats` | 訊息統計 |
| GET | `/permissions` | 取得 realtime RLS 權限（channel 訂閱／訊息發布權限） |

用途：管理 realtime pub/sub channel、訊息歷史、存取權限。實際 client 端訂閱走 Socket.IO（`SocketManager`），不透過本 REST domain。

---

## 17. `email` domain — `/api/email`

檔案：`backend/src/api/routes/email/index.routes.ts`。

| Method | Path | 認證 | 說明 |
|---|---|---|---|
| POST | `/send-raw` | `verifyUser`（另在 handler 內擋掉 `role === 'anon'`） | 寄送自訂 raw email（to/subject/body） |

用途：供已登入使用者（非匿名）呼叫的交易型郵件發送 API；與 `auth` domain 內建的驗證信/重設密碼信是不同的用途（那些是系統模板信）。

---

## 18. `deployments` domain — `/api/deployments`（Vercel-backed 站台部署）

檔案：`index.routes.ts` + `env-vars.routes.ts`（掛載於 `/env-vars`）。**全部 `verifyAdmin`**。Rate limiter：`deploymentsWriteLimiter`（大部分寫入端點；檔案內容串流 PUT 與部署狀態同步 POST 明確排除，因為串流/輪詢式呼叫頻率天生較高）。

代表性端點：

| Method | Path | Rate limiter | 說明 |
|---|---|---|---|
| POST | `/` | `deploymentsWriteLimiter` | 建立部署紀錄（舊版 zip 上傳流程） |
| POST | `/direct` | `deploymentsWriteLimiter` | 建立直接上傳式部署 |
| PUT | `/:id/files/:fileId/content` | 無（排除） | 串流單一部署檔案內容到 Vercel |
| POST | `/:id/start` | `deploymentsWriteLimiter` | 啟動部署 |
| GET | `/` | 無 | 列出部署（分頁） |
| GET | `/metadata` | 無 | 目前部署＋網域 URL |
| GET / POST / DELETE | `/domains[/:domain]`、`/domains/:domain/verify` | `deploymentsWriteLimiter`（寫入） | 自訂網域 CRUD／驗證 |
| POST | `/:id/sync` | 無（排除） | 同步部署狀態 |
| POST | `/:id/cancel` | `deploymentsWriteLimiter` | 取消部署 |
| GET / POST / DELETE | `/env-vars[/:id]` | — | Vercel 環境變數管理 |

用途：Vercel 後端支援的站台部署生命週期、自訂網域、環境變數管理。

---

## 19. `schedules` domain — `/api/schedules`

檔案：`backend/src/api/routes/schedules/index.routes.ts`。**`router.use(verifyAdmin)` 全域套用**，無專屬 rate limiter。

| Method | Path | 說明 |
|---|---|---|
| GET | `/` | 列出排程 |
| GET / PATCH | `/config` | 讀取／更新保留設定 |
| GET | `/:id` | 取得排程 |
| GET | `/:id/logs` | 執行日誌 |
| DELETE | `/:id` | 刪除排程 |
| POST | `/` | 建立排程（cron job） |
| PATCH | `/:id` | 更新／切換排程 |

用途：cron 排程任務管理（建立/更新外部 cron job，典型用法是定時呼叫 edge function）。

---

## 20. `payments` domain — `/api/payments`（雙供應商：Stripe + Razorpay）

掛載結構：`payments/index.routes.ts` 掛 `/stripe`、`/razorpay` 兩個子路由群，各自再細分 `catalog`／`config` 子路由。無專屬 rate limiter（依賴 provider 端 429 處理，見 `.trace/_context/integrations.md` 第 2 節）。

### 20.1 Stripe（`/api/payments/stripe`）

| Method | Path | 認證 | 說明 |
|---|---|---|---|
| GET | `/status`、`/config` | `verifyAdmin` | 連線狀態／設定 |
| POST | `/sync` | `verifyAdmin` | 同步 Stripe 資料 |
| POST | `/:environment/checkout-sessions` | `verifyUser` | 建立 Checkout session |
| POST | `/:environment/customer-portal-sessions` | `verifyUser` | 建立 Customer Portal session |
| PUT / DELETE | `/:environment/config` | `verifyAdmin` | 更新／刪除環境設定 |
| POST | `/:environment/webhook` | `verifyAdmin` | 設定管理式 Stripe webhook |
| GET/POST/PATCH/DELETE | `/:environment/catalog/products[/:id]`、`/prices[/:id]` | `verifyAdmin` | 商品／價格目錄 CRUD |
| GET | `/:environment/transactions`、`/subscriptions`、`/customers` | `verifyAdmin` | 查詢交易／訂閱／客戶 |

### 20.2 Razorpay（`/api/payments/razorpay`）

結構對稱：`GET /status`／`/config`、`POST /sync`（`verifyAdmin`）；`POST /:environment/orders`、`/orders/verify`、`/subscriptions`、`/subscriptions/verify`、`/subscriptions/:id/{cancel,pause,resume}`（皆 `verifyUser`）；`/:environment/config`、`/catalog/items`、`/catalog/plans`、`/customers`、`/transactions`、`/subscriptions`（列表，`verifyAdmin`）。

用途：開發者自有金流（developer-owned keys，test/live 雙環境）——設定與目錄管理限 admin，checkout/order/subscription 操作開放給已登入終端使用者。Webhook 接收見第 25 節。

---

## 21. `compute` domain — `/api/compute/services`（Fly.io-backed 長駐容器）

檔案：`backend/src/api/routes/compute/services.routes.ts`。**全部 `verifyAdmin`**。Rate limiter：`computeWriteLimiter`（寫入）、`computeLogsRateLimiter`（日誌）。

| Method | Path | Rate limiter | 說明 |
|---|---|---|---|
| GET | `/`、`/:id` | 無 | 列出／取得服務 |
| POST | `/` | `computeWriteLimiter` | 建立服務紀錄 |
| POST | `/deploy` | `computeWriteLimiter` | 準備 DB 記錄＋Fly app（尚未起機器） |
| POST | `/:id/deploy-token` | `computeWriteLimiter` | 核發 Fly deploy token（cloud-managed 模式供 CLI 用） |
| PATCH | `/:id` | `computeWriteLimiter` | 更新服務（環境變數變更會在審計日誌中遮蔽） |
| DELETE | `/:id` | `computeWriteLimiter` | 刪除服務（快照保留在審計日誌） |
| POST | `/:id/stop`、`/:id/start` | `computeWriteLimiter` | 停止／啟動 |
| GET | `/:id/events` | 無 | Fly machine 生命週期事件 |
| GET | `/:id/logs` | `computeLogsRateLimiter` | 容器 stdout/stderr（`next_token` 分頁） |

用途：管理部署在 Fly.io 上的長駐運算服務，每個服務 scope 到單一 `projectId`。README 標註此功能為 "private preview"，但實作已完整（見 `.trace/_context/recon.md` 第 6 節落差觀察）。

---

## 22. `analytics` domain — `/api/analytics`（PostHog-backed，僅限 Cloud）

檔案：`backend/src/api/routes/analytics/index.routes.ts`。讀取端 `verifyUser`，刪除連線 `verifyAdmin`。無專屬 rate limiter。

| Method | Path | 認證 | 說明 |
|---|---|---|---|
| GET | `/connection`、`/dashboards`、`/summary`、`/events` | `verifyUser` | 連線狀態／dashboard／摘要／事件查詢 |
| DELETE | `/connection` | `verifyAdmin` | 刪除 PostHog 連線 |
| GET | `/web-overview`、`/web-stats`、`/trends`、`/retention`、`/recordings` | `verifyUser` | 各類分析報表 |
| POST | `/recordings/:id/share` | `verifyUser` | 產生 session recording 分享連結 |

僅在 `PROJECT_ID` 存在且非 `'local'` 時可用，否則回 501 `ANALYTICS_UNAVAILABLE`（見 `.trace/_context/integrations.md` 第 10 節）。

---

## 23. `webscraper` domain — `/api/webscraper`（Apify-backed，僅限 Cloud）

檔案：`backend/src/api/routes/webscraper/index.routes.ts`。**全部 `verifyAdmin`**（因為會曝露有效的 OAuth token，明確限管理員）。無專屬 rate limiter。

| Method | Path | 說明 |
|---|---|---|
| GET / DELETE | `/apify/connection` | 連線狀態／刪除連線 |
| GET | `/apify/token` | 取得即時 Apify OAuth token |
| GET | `/apify/runs`、`/apify/actors`、`/apify/datasets`、`/apify/data` | 瀏覽 runs／actors／datasets／最新資料 |

同 `analytics`，僅限 InsForge Cloud（非 self-host 功能），見 `.trace/_context/integrations.md` 第 9 節。

---

## 24. `advisor` domain — `/api/advisor`（資料庫健康／安全分析）

檔案：`backend/src/api/routes/advisor/index.routes.ts`。**全部 `verifyAdmin`**，無專屬 rate limiter。

| Method | Path | 說明 |
|---|---|---|
| POST | `/scan` | 觸發手動掃描，回傳 `scanId` |
| GET | `/latest` | 最近一次掃描摘要 |
| GET | `/issues` | 掃描發現（可依 `severity`: critical/warning/info、`category`: security/performance/health 篩選，分頁） |
| GET | `/suppressions` | 列出已抑制的發現（"Ignored" 檢視） |
| POST | `/suppressions` | 抑制某個發現（`scope`: instance/rule；`reason` enum；`reason=other` 時需附 `note`） |
| DELETE | `/suppressions/:id` | 復原（取消抑制） |

用途：類 linter 的資料庫顧問功能（安全／效能／健康檢查），採 scan → issues → suppression 工作流程，全程僅限 admin。

---

## 25. `webhooks` domain — `/api/webhooks`（不在標準 `apiRouter` 下，掛載於 app 層）

**特殊之處**：`server.ts:161` 用 `express.raw({ type: 'application/json' })` 掛在 JSON body-parser **之前**，保留原始 bytes 供簽章驗證；**無 `verify*` 系列認證 middleware**，改以各家 provider 的簽章驗證取代身份認證。

| Method | Path | 簽章 Header | 驗證方式 |
|---|---|---|---|
| POST | `/api/webhooks/stripe/:environment` | `stripe-signature` | Stripe SDK 驗證簽章（覆蓋原始 body），失敗轉 400 `INVALID_INPUT`（`normalizeStripeWebhookError`） |
| POST | `/api/webhooks/razorpay/:environment` | `x-razorpay-signature`（另讀 `x-razorpay-event-id` 做冪等） | Razorpay 簽章驗證 |
| POST | `/api/webhooks/vercel` | `x-vercel-signature` | HMAC-SHA1，`crypto.timingSafeEqual` 常數時間比對，密鑰來自 `VERCEL_WEBHOOK_SECRET` |

用途：接收 Stripe／Razorpay／Vercel 的事件回呼，更新金流／部署狀態並透過 Socket.IO 廣播。詳見 `.trace/_context/integrations.md` 第 2 節。

---

## 26. `s3-gateway` domain — `/storage/v1/s3`（不在 `/api` 或標準 apiRouter 下）

檔案：`backend/src/api/routes/s3-gateway/{index.routes.ts, dispatch.ts, commands/*}`。**完整 AWS S3 協定相容 gateway**，認證是自訂 SigV4 middleware（`s3Sigv4Middleware`，`backend/src/api/middlewares/s3-sigv4.ts`），**不是** `verify*` 家族的一員——驗證的是 AWS Signature Version 4 請求簽章。僅在 `StorageService.isS3Provider()`（即設定了 `AWS_S3_BUCKET` 等 S3 相容後端）時才會掛載此路由。

依 method/path/query/header 由 `dispatchOp()` 分派到對應 S3 操作（`commands/` 下每個操作一個檔案）：

| S3 操作 | 對應 HTTP | 觸發條件 |
|---|---|---|
| ListBuckets | `GET /` | — |
| CreateBucket / DeleteBucket / HeadBucket | `PUT` / `DELETE` / `HEAD /:bucket` | — |
| ListObjectsV2（或 GetBucketLocation/Versioning/Cors） | `GET /:bucket` | 依 query flag 分流 |
| PutObject（或 CopyObject／PutObjectTagging） | `PUT /:bucket/:key` | `x-amz-copy-source` header 或 `?tagging` |
| GetObject（或 GetObjectTagging／ListParts） | `GET /:bucket/:key` | `?tagging` 或 `?uploadId` |
| DeleteObject（或 AbortMultipartUpload／DeleteObjectTagging） | `DELETE /:bucket/:key` | `?uploadId` 或 `?tagging` |
| DeleteObjects（批次） | `POST /:bucket?delete` | — |
| CreateMultipartUpload / CompleteMultipartUpload | `POST /:bucket/:key?uploads` / `?uploadId=` | — |
| UploadPart | `PUT /:bucket/:key?uploadId=&partNumber=` | — |

另會依 `maxS3UploadSize` 檢查 `x-amz-decoded-content-length`/`content-length`，並將內部錯誤轉換為標準 S3 XML 錯誤格式（`sendS3Error`），含針對 chunked-upload 簽章失敗的 `SignatureDoesNotMatch` 轉譯。

用途：讓標準 S3 client（`aws-sdk`、`s3cmd`、`rclone` 等）用原生 S3 REST API + SigV4 直接操作 InsForge storage，作為 REST-API 之外**第三種 API surface**（與 app REST API、PostgREST 並列）。

---

## 27. 跨 domain 觀察小結

1. **三種「直通式」API surface 並存**：(a) app 層 REST API（本文件主體）、(b) PostgREST 直通層（Part 1 第 4 節）、(c) S3-gateway（本節 26）。三者共用同一份底層資料（Postgres／物件儲存），但認證機制、錯誤格式、query 語法各自獨立。
2. **`verifyAdmin` 是絕對多數 domain 的預設值**：22 個 domain 中，`logs`／`metadata`／`secrets`／`realtime`／`schedules`／`webscraper`／`advisor`／`s3-access-key 管理` 等幾乎整個 domain 全數 `verifyAdmin`，反映這些功能本質上是「Dashboard／專案擁有者管理介面」，而非給終端使用者的資料 API。
3. **僅 `memory` domain 整體用 `verifyApiKey`**（非 `verifyAdmin` 也非 `verifyUser`），定位為 CLI/agent 專用、不透過使用者 JWT 存取的平台原語。
4. **`verifyCloudBackend` 極少使用**（僅見於 `usage/stats`），是唯一「InsForge Cloud 後端 → 單一 self-host/cloud project」方向的 backend-to-backend 認證管道，透過 JWKS 驗證。
5. **Cloud-only 功能透過執行期檢查而非路由層隔離**：`analytics`／`webscraper` 在程式碼層檢查 `projectId !== 'local'`，而非在路由掛載時就地排除；`database/backups` 則是路由掛載時期用 `isCloudEnvironment()` 直接不掛載（self-host 才有 `/backups`，cloud 環境下該路徑不存在）。

---

（Part 1 見 `API_SURFACE_part1.md`）
