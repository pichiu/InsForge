# 外部整合（External Integrations）盤點

範圍：`backend/src/providers/` 下所有第三方 API/SDK/服務包裝層，及相關的 webhook 驗證、rate-limit、retry 邏輯。
產出時間：2026-07-11。中間工作文件，力求完整而非精美。

---

## 1. Storage — AWS S3（含 local-disk fallback）

**用途**：物件儲存（bucket/key 模型），供 Storage API 使用；支援 presigned upload/download URL、CloudFront 簽名 URL、S3-compatible 服務（MinIO/Wasabi/Tencent COS/Aliyun OSS）。

**SDK 包裝位置**：
- `backend/src/providers/storage/s3.provider.ts` — `S3StorageProvider` class（第 60 行起）
  - 使用 `@aws-sdk/client-s3`（`PutObjectCommand`/`GetObjectCommand`/`DeleteObjectCommand`/`ListObjectsV2Command`/`DeleteObjectsCommand`/`HeadObjectCommand`/`CopyObjectCommand`/multipart 系列，第 1-14 行 import）
  - Presigned URL：`@aws-sdk/s3-request-presigner`（`getSignedUrl`，第 16 行）、`@aws-sdk/s3-presigned-post`（`createPresignedPost`，第 17 行）
  - CloudFront 簽名：`@aws-sdk/cloudfront-signer`（`getSignedUrl as getCloudFrontSignedUrl`，第 18 行）
- `backend/src/providers/storage/base.provider.ts` — `StorageProvider` interface（provider 合約）
- `backend/src/providers/storage/local.provider.ts` — `LocalStorageProvider`（本地檔案系統 fallback，第 25 行起）
- Provider 選擇邏輯：`backend/src/services/storage/storage.service.ts:44-65` — 建構子中判斷 `appConfig.storage.s3Bucket` 是否存在：有值則用 `S3StorageProvider`（第 53-60 行），否則 fallback 到 `LocalStorageProvider`（第 64 行，使用 `appConfig.storage.storageDir`）。註解明確指出「local installs aren't branched」，即本地模式不支援 branch fallback。

**下載策略（CloudFront → S3 presigned → local）**：`s3.provider.ts:429-525`
- 若 `AWS_CLOUDFRONT_URL` 設定且未使用自訂 S3 endpoint，優先用 CloudFront 簽名 URL（第 436-490 行）。
- 若 CloudFront key pair ID 或 private key 缺失，記錄 warning 並 fallback 回 S3 presigned URL（第 442-444 行「CloudFront URL configured but missing key pair ID or private key, falling back to S3」）。
- CloudFront 簽名產生失敗（try/catch，第 446-491 行）同樣 fallback 到 S3 presigned URL（註解：「Fall through to S3 signed URL generation」）。
- S3 presigned URL 有效期：public 檔案 7 天（`SEVEN_DAYS_IN_SECONDS`，第 52 行），private 檔案預設 1 小時（`ONE_HOUR_IN_SECONDS`，第 51 行）。SigV4 簽名涵蓋所有 query 參數，因此 `?v=<version>` cache-busting 參數不會附加在 S3 presigned URL（僅 CloudFront 支援，因為 CloudFront canned-policy 簽名排除該三個保留參數）。

**Branch 模式 fallback（多租戶/分支專案）**：`s3.provider.ts:120-140` `withFallback()` — 讀取路徑先試 branch 的 S3 key，若回傳 null（404 miss）才 fallback 到 parent 的 S3 key（`PARENT_APP_KEY`）；寫入路徑永不 fallback 到 parent（只寫 branch 路徑）。`isS3NotFound()`（第 37-49 行）統一處理不同 S3-compatible 後端的 404 語意差異（`NotFound`/`NoSuchKey`/`Code===NoSuchKey`/`httpStatusCode 404`）。

**失敗處理**：
- `putObject`（第 148-166 行）：try/catch，記錄 `logger.error('S3 Upload error', ...)` 後 rethrow，無 retry。
- `getObject`（第 168-188 行）：任何錯誤最終被吞掉、回傳 `null`（保留舊有 service 層行為），只有透過 `withFallback` 的 404-only fallback 觸發 parent 查詢，非 404 錯誤會直接 propagate 並在外層被吞成 null。
- 未見任何 retry-with-backoff 或 circuit breaker，僅有單次 try/catch + fallback 鏈（CloudFront → S3 presign）與 branch fallback（S3 key → parent S3 key）。

**環境變數**（`.env.example`）：
`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_REGION` / `AWS_S3_BUCKET`（AWS 憑證，也作為 CloudWatch fallback）；`S3_ACCESS_KEY_ID` / `S3_SECRET_ACCESS_KEY` / `S3_ENDPOINT_URL` / `S3_FORCE_PATH_STYLE`（S3-compatible，優先權高於 `AWS_*`）；`AWS_CLOUDFRONT_URL` / `AWS_CLOUDFRONT_KEY_PAIR_ID` / `AWS_CLOUDFRONT_PRIVATE_KEY`（CloudFront 簽名，僅原生 AWS S3 適用，設定 `S3_ENDPOINT_URL` 時會被忽略）；`STORAGE_DIR`（local fallback 目錄）；`MAX_FILE_SIZE`。

---

## 2. Payments — Stripe + Razorpay

**用途**：處理商戶自有金流（developer-owned keys，test/live 雙環境）：客戶、商品、訂閱、checkout、customer portal、webhook 事件同步。

**SDK 包裝位置**：
- Stripe：`backend/src/providers/payments/stripe.provider.ts` — `StripeProvider` class（第 87 行起），使用官方 `stripe` npm SDK（`import Stripe from 'stripe'`，第 1 行），建構子第 94-99 行 `new Stripe(secretKey, { typescript: true })`。
- Razorpay：`backend/src/providers/payments/razorpay.provider.ts` — 使用官方 `razorpay` npm SDK（第 1 行 `import Razorpay from 'razorpay'`）。
- 金鑰前綴驗證：`stripe.provider.ts:30-35`（`validateStripeSecretKey`，要求 `sk_live_`/`sk_test_` 前綴）、`razorpay.provider.ts:17-23`（`validateRazorpayKey`，要求 `rzp_live_`/`rzp_test_` 前綴）。金鑰遮罩：`maskStripeKey`（第 37-46 行）、`maskRazorpayKey`（第 26-34 行）。

**Webhook 簽章驗證**（`backend/src/api/routes/webhooks/`）：
- Stripe：`webhooks/stripe.routes.ts` — `POST /api/webhooks/stripe/:environment`（第 23 行起）。要求 `stripe-signature` header（第 31-34 行），要求 raw Buffer body（第 36-40 行），交由 `StripeWebhookService.handleStripeWebhook()` 用 Stripe SDK 驗證簽章；驗證失敗轉換為 `AppError(400, INVALID_INPUT)`（`normalizeStripeWebhookError`，第 9-14 行，比對 `error.name === 'StripeSignatureVerificationError'`）。
- Razorpay：`webhooks/razorpay.routes.ts` — `POST /api/webhooks/razorpay/:environment`（第 10 行起）。要求 `x-razorpay-signature` header（第 14-17 行）與 raw Buffer body（第 19-22 行），另外讀取 `x-razorpay-event-id` 做冪等/去重（第 24 行）。
- Vercel（附帶對照）：`webhooks/vercel.routes.ts:25-64` — HMAC-SHA1 於 `x-vercel-signature` header，使用 `crypto.timingSafeEqual` 做 constant-time 比對（第 53-61 行），防 timing attack。

**失敗處理**：
- `backend/src/providers/payments/stripe-errors.ts` — `normalizeStripeError()`（第 33-53 行）依 Stripe 錯誤類型分流：`StripeRateLimitError` 或 HTTP 429 → `AppError(429, RATE_LIMITED)`；`StripeAuthenticationError`/`StripePermissionError` → `AppError(status, PAYMENT_CONFIG_INVALID)`；`StripeCardError` → `AppError(status, PAYMENT_METHOD_DECLINED)`；其餘包成 `UpstreamError`。無自動 retry（Stripe SDK 本身內建 retry 行為未見顯式覆寫 ⚠️ 未驗證是否使用 SDK 預設 `maxNetworkRetries`）。
- `backend/src/providers/payments/razorpay-errors.ts` — 對應的 Razorpay 錯誤正規化（結構與 Stripe 版本相同模式）。
- 未發現 circuit breaker；webhook 端點對簽章錯誤一律回 4xx，不重試（重試由 Stripe/Razorpay 端依其自身 webhook retry 政策負責）。

**環境變數**：`STRIPE_LIVE_SECRET_KEY` / `STRIPE_TEST_SECRET_KEY`；`RAZORPAY_LIVE_KEY_ID` / `RAZORPAY_LIVE_KEY_SECRET` / `RAZORPAY_TEST_KEY_ID` / `RAZORPAY_TEST_KEY_SECRET`。

---

## 3. Email — nodemailer (SMTP) + Cloud Email Provider

**用途**：交易型郵件（驗證信、重設密碼等 template email）與自訂 raw email。

**SDK 包裝位置**：
- `backend/src/providers/email/smtp.provider.ts` — `SmtpEmailProvider`，使用 `nodemailer`（第 1 行 `import nodemailer from 'nodemailer'`）。`createTransporter()`（第 33-40 行）：`connectionTimeout: 10000`（10 秒連線逾時），`secure: config.port === 465`。
- `backend/src/providers/email/base.provider.ts` — `EmailProvider` interface（`sendWithTemplate` / `sendRaw` / `supportsTemplates`）。
- `backend/src/providers/email/cloud.provider.ts` — `CloudEmailProvider`，走 InsForge Cloud backend 代理（非直接對接第三方郵件服務），用短效 JWT（`expiresIn: '10m'`，第 40-42 行）簽署後打 axios 到 cloud API；缺少 `PROJECT_ID` 或 `JWT_SECRET` 時直接丟 `AppError(500, INTERNAL_ERROR)`（第 22-35 行）。
- `backend/src/services/email/` 下有 `smtp-config.service.ts`、`email-template.service.ts`（HTML 模板渲染，含 `escapeHtml`/`escapeRegex` 防注入，`smtp.provider.ts:11-20`）。

**失敗處理**：`smtp.provider.ts` 的 `send()` 方法（第 74 行起，⚠️ 未讀取完整內文）具備 try/catch 與 transporter cleanup（依 docstring 描述）；`getRequiredConfig()`（第 60-70 行）在 SMTP 未設定/未啟用時丟 `AppError(500, EMAIL_SMTP_CONNECTION_FAILED)`。未見 retry-with-backoff 或 circuit breaker，僅單次嘗試 + 明確錯誤碼。link 欄位有防護：非 `http(s)://` 開頭的連結值會被拒絕並替換為 `#`（第 45-49 行，防止 email 樣板注入惡意連結）。

**環境變數**：`.env.example` 未見獨立的 `SMTP_*` 變數區塊（⚠️ 未驗證，SMTP 設定可能透過 dashboard/`SmtpConfigService` 存於資料庫而非環境變數，而非 `.env.example`）。

---

## 4. OAuth Providers — Google/GitHub/Discord/Microsoft/LinkedIn/X/Apple/Facebook/Custom

**用途**：第三方社交登入（authorization code exchange → user profile fetch）。

**SDK 包裝位置**：全部位於 `backend/src/providers/oauth/`，一律用 `axios` 直接呼叫各家 OAuth2 REST API（無官方 SDK），共同介面見 `oauth/base.provider.ts`（`OAuthProvider`）：
- Google：`google.provider.ts` — token exchange `axios.post('https://oauth2.googleapis.com/token', ...)`（第 132 行）。
- GitHub：`github.provider.ts` — `axios.get('https://api.github.com/user', ...)`（第 147 行）、`.../user/emails`（第 157 行，用於取得未公開但已驗證的 email）。
- Discord：`discord.provider.ts` — `axios.get('https://discord.com/api/users/@me', ...)`（第 149 行）。
- Microsoft：`microsoft.provider.ts` — `axios.get('https://graph.microsoft.com/v1.0/me', ...)`（第 174 行，Microsoft Graph API）。
- LinkedIn：`linkedin.provider.ts`（token exchange 第 128 行）。
- X (Twitter)：`x.provider.ts` — `axios.post('https://api.twitter.com/2/oauth2/token', ...)`（第 132 行）、`axios.get('https://api.twitter.com/2/users/me', ...)`（第 151 行）。
- Apple：`apple.provider.ts` — `axios.post('https://appleid.apple.com/auth/token', ...)`（第 180 行），Sign in with Apple 特有的 JWT client secret 需自行組裝（依 `.env.example` 說明，`APPLE_CLIENT_SECRET` 是 JSON：teamId/keyId/privateKey）。
- Facebook：`facebook.provider.ts` — Graph API v21.0（`https://graph.facebook.com/v21.0/oauth/access_token` 第 113 行，`.../v21.0/me` 第 144 行）。⚠️ 未在 `.env.example` 中見到對應 `FACEBOOK_CLIENT_ID/SECRET`，可能為程式碼支援但未在文件中曝光的功能。
- Custom（OIDC）：`custom.provider.ts` — 支援自訂 OIDC discovery endpoint（`axios.get<DiscoveredEndpoints>(config.discoveryEndpoint, ...)` 第 45 行）供任意 OIDC provider 接入。

**失敗處理**：每個 provider 的 token exchange / profile fetch 都各自 try/catch，用 `axios.isAxiosError(error) && error.response` 判斷後拋出對應的 `AppError`（模式一致，散落在各檔案的第 130-200 行區間）。未見統一的 retry 或逾時設定（⚠️ 未驗證各 axios 呼叫是否設有 timeout；至少在抓取的片段中未見顯式 `timeout` 參數，可能依賴 axios 預設無限等待）。無 circuit breaker。

**環境變數**：`GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET`、`GITHUB_CLIENT_ID`/`GITHUB_CLIENT_SECRET`、`MICROSOFT_CLIENT_ID`/`MICROSOFT_CLIENT_SECRET`、`DISCORD_CLIENT_ID`/`DISCORD_CLIENT_SECRET`、`LINKEDIN_CLIENT_ID`/`LINKEDIN_CLIENT_SECRET`、`X_CLIENT_ID`/`X_CLIENT_SECRET`、`APPLE_CLIENT_ID`/`APPLE_CLIENT_SECRET`。Redirect URI 慣例為 `http://localhost:7130/auth/<provider>/callback`。

---

## 5. AI / Model Gateway — OpenRouter（OpenAI-compatible）

**用途**：AI 功能的模型閘道（chat completion、usage/quota 查詢），透過 OpenRouter 統一多家 LLM 供應商。

**SDK 包裝位置**：`backend/src/providers/ai/openrouter.provider.ts` — `OpenRouterProvider`（singleton，第 84 行起）。
- 使用官方 `openai` npm SDK 當作 OpenAI-compatible client，但改指向 OpenRouter：`baseURL: 'https://openrouter.ai/api/v1'`（第 107 行）。
- 額度/用量查詢直接用原生 `fetch`：`fetch('https://openrouter.ai/api/v1/key', ...)`（第 250 行）、`https://openrouter.ai/api/v1/activity`（第 288 行）。
- API Key 來源分雙軌：cloud 模式向 InsForge cloud-backend 換取代管 key（`CloudCredentialsResponse`，第 9-16 行），或直接使用環境變數 `OPENROUTER_API_KEY`（`ApiKeySource = 'cloud' | 'env'`，第 55 行）。cloud key 有 `fetchPromise`/`rotationPromise` 兩個 in-flight promise 去重機制（第 90-91 行）避免併發重複換 key。

**失敗處理**：
- 第 266 行：偵測到 rate limit 時丟出「OpenRouter rate limit exceeded. Please wait before retrying.」訊息的錯誤（未見自動 retry，僅提示使用者手動重試）。
- 第 711-716 行：對一般模型 provider 的 429 也回傳提示「Wait a moment and retry, or check your API key rate limits.」。
- 未見 retry-with-backoff 或 circuit breaker 的實作，錯誤直接以 `UpstreamError`/`AppError` 往外拋，由呼叫端（如 dashboard）決定是否重試。

**環境變數**：`OPENROUTER_API_KEY`、`MAX_COMPLETION_TOKENS`（預設 16384，非正整數會被靜默忽略並使用預設值）。

---

## 6. Compute — Fly.io

**用途**：使用者自訂運算環境（Docker container）部署與生命週期管理（machines API）。

**SDK 包裝位置**：`backend/src/providers/compute/fly.provider.ts` — `FlyProvider`（singleton，第 55 行起，`implements ComputeProvider`）。無官方 SDK，直接用原生 `fetch` 呼叫兩個不同 host：
- Machines API：`FLY_API_BASE = 'https://api.machines.dev/v1'`（第 10 行）。
- Logs API（未官方文件化但穩定的端點）：`FLY_LOGS_API_BASE = 'https://api.fly.io/api/v1'`（第 18 行）；程式碼註解明確指出這是不同 host + 不同 auth scheme（`FlyV1 <macaroon>` 而非 `Bearer`），且「Fly documents this endpoint as stable-but-unofficial」（第 11-17 行）。
- 統一請求包裝：`request<T>()`（第 84-100 行）與 `requestJson<T>()`（第 102-108 行），非 2xx 回應丟出自訂 `FlyHttpError`（帶 HTTP status，第 44-52 行），供上層區分 404（機器已消失）與其他暫時性錯誤。

**失敗處理 / Retry**：
- IP 配置 race condition retry：`allocatePublicIps`/`allocateOneIp`（第 128-140 行起）——Fly 的 GraphQL 在剛建立 app 後可能回傳 `{ipAddress: null}` 而非錯誤，因此用短暫 backoff 重試直到拿到真正的 IP（避免 app 只有 IPv6，IPv4-only client 解析失敗）。
- 狀態輪詢 retry：`waitForState()`（第 380-406 行）——每 1 秒輪詢一次機器狀態，最長 `timeoutMs`（預設 30000ms）。遇到 `MachineGoneError`（確定性 404）立即 fail fast（第 396-399 行）；其他暫時性錯誤記錄 warning 後繼續重試（第 400 行「Transient error polling machine state, retrying」），逾時後拋出 `Machine did not reach state [...] within Nms` 錯誤。
- 外部請求逾時：第 496 行 `signal: AbortSignal.timeout(15_000)`（15 秒逾時，用於某個對外請求 ⚠️ 未逐行核對具體是哪個 API 呼叫）。
- 未見 circuit breaker。

**環境變數**：`FLY_API_TOKEN`、`FLY_ORG`（兩者皆須設定，`isConfigured()` 第 68-70 行同時檢查）、`COMPUTE_DOMAIN`（選填，自訂 wildcard domain）。Cloud 模式則透過 `PROJECT_ID` + `CLOUD_API_HOST` 自動偵測，走 `CloudComputeProvider`（不同於本節的 self-host `FlyProvider`，⚠️ 該檔案 `compute.provider.ts`/`cloud.provider.ts` 未逐行閱讀）。

---

## 7. Deployment — Vercel + Deno Deploy (Subhosting)

### 7a. Vercel

**用途**：自架站台部署與自訂網域管理（legacy site deployment 路徑）。

**SDK 包裝位置**：`backend/src/providers/deployments/vercel.provider.ts`（1278 行，無官方 SDK，全用 `axios`，第 1 行 import）。

**Rate-limit retry（顯式實作）**：
- `withVercelRateLimitRetry()`（第 133-181 行）：包裝可重試的寫入類 API 呼叫（`createDeployment`/`cancelDeployment`/`upsertEnvironmentVariables`/`addCustomDomain`/`removeCustomDomain`/`verifyCustomDomain`，見第 128-131 行註解列表）。邏輯：偵測 axios 429 → 讀取 `X-RateLimit-Reset`（Unix epoch 秒）算出等待時間，若 header 缺失則用 `2^attempt * baseDelayMs` 指數 backoff + jitter，並以 `maxDelayMs` 封頂（第 160-176 行）。重試次數用盡後拋出 `AppError(429, RATE_LIMITED)`（第 152-157 行）而非讓原始 axios error 外洩。
- 檔案上傳的獨立 retry loop（`uploadFile`，第 1042-1076 行，「keeps its own bespoke streaming-aware retry loop」，未走 `withVercelRateLimitRetry`），常數見第 14-19 行：`UPLOAD_MAX_RETRIES=3`、`UPLOAD_BACKOFF_BASE_MS=1000`、`UPLOAD_BACKOFF_MAX_MS=30000`、`UPLOAD_JITTER_MAX_MS=500`、`UPLOAD_BATCH_SIZE=5`、`UPLOAD_INTER_BATCH_DELAY_MS=200`。
- 逾時：`VERCEL_UPLOAD_TIMEOUT_MS = 120_000`（120 秒，第 12 行），用於 `timeout`（第 1121 行）與 `timeoutMs`（第 1144 行）。第 236 行另有針對 `ECONNABORTED`/訊息含 `timeout` 的錯誤判斷邏輯。
- 缺少 `VERCEL_TOKEN` 時明確報錯（第 260 行「VERCEL_TOKEN not found in environment variables」）。

**環境變數**：`VERCEL_TOKEN`、`VERCEL_TEAM_ID`、`VERCEL_PROJECT_ID`。註解指出「Legacy deployments also require AWS_S3_BUCKET to be configured」，顯示 Vercel 部署路徑與 S3 storage 有耦合。

### 7b. Deno Deploy / Subhosting

**用途**：Serverless Functions（Edge functions）託管，取代/補充 Vercel 部署路徑，是目前主要的 functions 部署後端。

**SDK 包裝位置**：`backend/src/providers/functions/deno-subhosting.provider.ts` — `DenoSubhostingProvider`（第 290 行起）。API base：`DENO_SUBHOSTING_API_BASE = 'https://api.deno.com/v2'`（第 25 行）。
- `isConfigured()`（第 305-308 行）：需同時有 `DENO_DEPLOY_TOKEN` 與 `DENO_DEPLOY_ORG_ID`（透過 `appConfig.denoSubhosting.token`/`organizationId`）。

**Fetch wrapper（含 timeout + 雙層 retry）**：`fetchWithRetry`（第 55 行起，docstring：「Fetch with timeout, retry for transient network errors (DNS/socket), and a separate retry layer for HTTP 429 (rate-limited) responses」）：
- 逾時層：`DEFAULT_TIMEOUT_MS`（第 70 行參數預設），用 `Promise.race` + `setTimeout` reject 實作請求逾時（第 78-98 行）。
- 429 retry 層：讀取 `retry-after` header 計算 `retryAfterMs`，clamp 在 `rateLimitBackoffMs[r]`（依重試次數的預設 backoff 陣列）與 `MAX_RETRY_AFTER_MS`（第 34 行常數，防止上游回傳異常大值卡住 worker）之間（第 109-133 行）；重試用盡後拋 `AppError(429, ..., 'Deno Deploy rate limit exceeded after retries. Please retry shortly.')`（第 143-149 行）。
- 一般傳輸錯誤（DNS/socket）另有短暫等待後重試（第 154-176 行，「Wait briefly before retry」）。

**呼叫端整合**：`backend/src/services/functions/function.service.ts`
- `isSubhostingConfigured()`（第 681-683 行）→ 直接委派 `this.denoSubhostingProvider.isConfigured()`。
- `getDeploymentUrl()`（第 733-757 行）→ 先查記憶體快取 `this.cachedDeploymentUrl`，miss 則查 DB（`functions.deployments` 表，`status='success' AND url IS NOT NULL`，取最新一筆並寫回快取）；查詢失敗僅記錄 `logger.error` 並回傳 `null`（fail-open，不拋錯）。
- `syncDeployment()`（第 758 行起）：啟動時若 Deno Deploy 未設定則直接 `logger.debug` 略過（第 767-770 行）；若已有成功部署則跳過重新部署（第 772-777 行）。

**環境變數**：`DENO_DEPLOY_TOKEN`、`DENO_DEPLOY_ORG_ID`。

---

## 8. Logs — AWS CloudWatch Logs（含 local-file fallback）

**用途**：InsForge 各元件（insforge/postgREST/postgres/function）的集中式日誌查詢與統計。

**SDK 包裝位置**：`backend/src/providers/logs/cloudwatch.provider.ts` — `CloudWatchProvider extends BaseLogProvider`（第 14 行起），使用 `@aws-sdk/client-cloudwatch-logs`（`CloudWatchLogsClient`/`DescribeLogStreamsCommand`/`FilterLogEventsCommand`/`StartQueryCommand`/`GetQueryResultsCommand`/`CreateLogGroupCommand`，第 1-8 行 import）。
- `initialize()`（第 20-49 行）：log group 命名規則 `/insforge/${PROJECT_ID}`（可用 `CLOUDWATCH_LOG_GROUP` 覆寫），region 預設 `us-east-2`（`AWS_REGION`/`AWS_DEFAULT_REGION` fallback）。啟動時嘗試 `CreateLogGroupCommand`，若已存在（`ResourceAlreadyExistsException`）僅記 info log（第 41-48 行），非該例外則記 warning 但不中斷啟動（fail-open）。
- Provider 選擇：`backend/src/services/logs/log.service.ts:45-48` — 依設定選 `CloudWatchProvider`（第 45 行）或 `LocalFileProvider`（第 48 行，本地檔案 fallback，位於 `backend/src/providers/logs/local.provider.ts`）。`.env.example` 註解明確：「CloudWatch Logging: Set AWS_REGION + AWS credentials above / File-based Logging: Configure LOGS_DIR below (used when AWS credentials not provided)」。

**失敗處理**：查詢方法（第 66/103/291/435 行附近）遇錯直接丟 `AppError`，未見顯式 retry-with-backoff 或 circuit breaker（AWS SDK v3 client 本身內建有限次數的 transient retry，⚠️ 未驗證是否有自訂覆寫 `maxAttempts`）。

**環境變數**：`AWS_REGION`（或 `AWS_DEFAULT_REGION`）、`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`（與 storage 共用）、`CLOUDWATCH_LOG_GROUP`（選填覆寫）、`PROJECT_ID`（用於 log group 命名）、`CW_SUFFIX_*` 系列（各元件 log stream 後綴，程式內預設值，未見於 `.env.example`，⚠️ 未驗證是否曝光給使用者設定）、`LOGS_DIR`（local fallback）。

---

## 9. Web Scraping — Apify（透過 cloud-backend 代理）

**用途**：網頁爬蟲 / actor 執行與資料集查詢，僅限 InsForge Cloud（非 self-host 功能）。

**SDK 包裝位置**：`backend/src/providers/webscraper/apify.provider.ts` — `ApifyProvider`（singleton，第 34 行起）。**並非直接呼叫 Apify 官方 API**，而是用 `axios`（第 2 行）搭配短效 JWT（`jwt.sign({ sub: projectId }, secret, { expiresIn: '10m' })`，第 68 行）代理呼叫 InsForge cloud-backend 的 project-JWT 路由（程式註解：「Mirrors PostHogProvider: signs a short-lived project JWT and proxies to cloud-backend's project-JWT routes」，第 8-9 行）。
- `isEnabled()`（第 45-47 行）：僅當 `appConfig.cloud.projectId` 存在且非 `'local'` 時才可用；否則 `throwUnsupported()`（第 49-55 行）丟出 `AppError(501, INTERNAL_ERROR, 'Apify integration is only available on Insforge Cloud, not in self-hosted mode.')`。
- 回應以 zod schema 驗證（`apifyConnectionSchema`/`apifyTokenSchema`/`apifyRunsSchema`/`apifyActorsSchema`/`apifyDatasetsSchema`/`apifyLatestDataSchema`，第 9-32 行），Apify 的 run/dataset item 內容型別保留 `unknown`（只驗證外層 envelope）。

**失敗處理**：signToken 缺少 `PROJECT_ID`/`JWT_SECRET` 時直接 throw（第 60-71 行）。⚠️ 未見該 provider 檔案中的 retry 邏輯（213 行檔案內僅讀了前 80 行，其餘方法未逐一核對）。

**環境變數**：本身不需要 self-host 端的 Apify API key（因為是 cloud-backend 代理），依賴 `PROJECT_ID`/`JWT_SECRET`（用於簽署代理請求）。

---

## 10. Analytics — PostHog（透過 cloud-backend 代理）

**用途**：產品分析（dashboards、events、web overview/stats、trends、retention、session recordings），僅限 InsForge Cloud。

**SDK 包裝位置**：`backend/src/providers/analytics/posthog.provider.ts` — `PostHogProvider`（singleton，第 29 行起）。與 Apify 相同模式：不直接呼叫 PostHog API，而是用 `axios`（第 2 行）+ 短效 JWT（`expiresIn: '10m'`，第 65 行）代理到 cloud-backend 的 `/projects/v1/${projectId}/posthog/...` 路由（`url()` helper，第 73-75 行）。
- `isEnabled()`（第 38-40 行）同樣僅在 `projectId` 存在且非 `'local'` 時可用，否則 `AppError(501, ANALYTICS_UNAVAILABLE, ...)`（第 42-48 行）。
- 逾時：`getConnection()` 中 axios 呼叫帶 `timeout: 10000`（10 秒，第 85-88 行）——是本檔案中少數明確設定 timeout 的呼叫。
- 回應以大量 zod schema 驗證（`posthogConnectionSchema`/`posthogDashboardsResponseSchema`/`posthogSummarySchema`/`posthogEventsResponseSchema`/`posthogWebOverviewResponseSchema`/`posthogWebStatsResponseSchema`/`posthogTrendsResponseSchema`/`posthogRetentionResponseSchema`/`posthogRecordingsResponseSchema`/`posthogShareTokenResponseSchema`，第 5-16 行）。

**失敗處理**：⚠️ 未逐一核對每個 API 方法的完整 catch 區塊（僅讀取前 90 行），但至少 `getConnection()` 有 try/catch（第 83-91 行起）。未見 retry 或 circuit breaker。

**環境變數**：`VITE_PUBLIC_POSTHOG_KEY`（`.env.example` 標註「Only needed for local development」，即前端本地開發用的 PostHog project key，與後端代理路徑是分開的兩件事——後端走 cloud-backend 代理不需要此 key）。

---

## 11. Rate-limiting / Retry 機制總覽

**Express 層級 rate limiting**：`backend/src/api/middlewares/rate-limiters.ts`，使用 `express-rate-limit`（第 1 行 `import rateLimit from 'express-rate-limit'`）。重點限流器：
- `sendEmailOTPRateLimiter`（第 53-57 行）：15 分鐘窗口，每 IP 最多 5 次。
- `s3AccessKeyManagementRateLimiter`（第 79-81 行）：15 分鐘窗口，每 IP 最多 20 次。
- `computeLogsRateLimiter`（第 106-108 行）：1 分鐘窗口，每 IP 最多 120 次。
- `verifyOTPRateLimiter`（第 130-132 行）：15 分鐘窗口，每 IP 最多 10 次。
- 寫入端點動態限流器（第 420-433 行起）：5 分鐘窗口，`max` 依 `getWriteEndpointLimit(category)` 動態決定，並可透過 `isWriteRateLimitDisabled()`（第 234 行）整體停用（`skip: () => isWriteRateLimitDisabled()`）。

**各 provider 的自訂 retry-with-backoff 統整**（均為手寫 loop，非共用函式庫）：
| Provider | 位置 | 策略 |
|---|---|---|
| Vercel | `deployments/vercel.provider.ts:133-181`（API 呼叫）、`:1042-1076`（檔案上傳） | 429 專用，指數 backoff + jitter，讀取 `X-RateLimit-Reset` |
| Deno Deploy | `functions/deno-subhosting.provider.ts:55-176` | 逾時 + 429 backoff（讀 `retry-after`）+ 一般傳輸錯誤重試，三層合一 |
| Fly.io | `compute/fly.provider.ts:380-406`（狀態輪詢）、`:128-140`（IP 配置） | 固定間隔輪詢（1 秒）直到逾時；非 retry-on-error 意義上的 backoff |
| OpenRouter | `ai/openrouter.provider.ts:266,711-716` | 僅偵測並回報 429，不自動重試（丟給呼叫端/使用者） |

**Circuit breaker**：整個 `backend/src/providers/` 未發現任何 circuit breaker 實作或函式庫（如 `opossum`）的使用痕跡。⚠️ 未驗證：可能存在於未搜尋到的其他路徑（例如 middleware 或 infra 層），但以 provider 目錄為範圍未見。所有「失敗隔離」策略實質上是：(a) fallback chain（S3→local storage、CloudFront→S3 presign、CloudWatch→local file logs）、(b) 明確的 429 backoff retry（Vercel/Deno/CloudWatch 之外的三者），(c) fail-open 的 try/catch + 回傳 null/預設值（多見於非關鍵路徑，如 `getDeploymentUrl`、CloudWatch log group 建立）。

---

## 待確認事項（⚠️ 未驗證清單）

1. Stripe/Razorpay SDK 是否使用其 SDK 內建的 `maxNetworkRetries`/自動重試（未在 provider 建構子中看到顯式覆寫）。
2. OAuth providers（`oauth/*.ts`）的 axios 呼叫是否設有逾時；抽樣程式碼片段中未見 `timeout` 參數。
3. `SmtpEmailProvider.send()` 完整方法體（僅讀取方法簽章與註解，第 74 行起未完整核對 retry/cleanup 細節）。
4. `webscraper/apify.provider.ts` 除 `getConnection`/token 相關方法外，其餘方法（runs/actors/datasets 查詢）的錯誤處理與 retry 細節。
5. `compute/cloud.provider.ts`、`compute.provider.ts`（cloud-managed 模式的 compute provider，不同於 self-host `fly.provider.ts`）未逐行核對。
6. `CW_SUFFIX_*` 系列環境變數是否記錄於使用者可見文件（`.env.example` 中未見，僅在程式碼中作為 `process.env` 讀取的預設值）。
7. Facebook OAuth（`oauth/facebook.provider.ts`）對應的 `FACEBOOK_CLIENT_ID`/`FACEBOOK_CLIENT_SECRET` 未見於 `.env.example`，此功能是否對外可用尚待確認。
