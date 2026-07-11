# InsForge 擴充點（Extension Points）調查筆記

> 目的：找出 InsForge monorepo 中「設計給人在不動 core code 的情況下新增功能」的地方。
> 涵蓋 backend route/service 慣例、payments provider、oauth provider、Edge Functions、middleware chain、dashboard feature 模組、UI component library。
> 狀態：中間工作文件，重技術正確性、輕排版。

---

## 1. Backend route/service 擴充模式（新增一個 domain，例如 payments / webscraper / advisor）

### 1.1 觀察到的「新增模組」流程（reverse-engineered recipe）

以 `backend/src/api/routes/` 下 22 個 route group 為樣本，比對 `webscraper`（小型、單一 provider）與 `payments`（大型、多 provider）兩個案例，得出一致的四層結構：

```
backend/src/api/routes/<domain>/index.routes.ts   -- Express Router，掛 middleware + 呼叫 service
backend/src/services/<domain>/<domain>.service.ts -- 業務邏輯，singleton（getInstance()）
backend/src/providers/<domain>/<vendor>.provider.ts -- 外部 API/SDK 封裝（可選，只有需要對接第三方時才有）
backend/src/server.ts                              -- 手動 import router 並 apiRouter.use('/<domain>', xxxRouter)
```

具體證據：

- `backend/src/api/routes/webscraper/index.routes.ts:1-20` — 標準寫法：
  `import { WebscraperService } from '@/services/webscraper/webscraper.service.js'`，
  `const service = WebscraperService.getInstance()`，每個 handler 都是
  `router.<verb>('/path', verifyAdmin, async (req, res, next) => { try {...} catch(err){ next(err) } })`。
- `backend/src/services/webscraper/` 對應 service，`backend/src/providers/webscraper/apify.provider.ts` 封裝第三方 Apify API。
- 大型模組（payments）走同一套但巢狀更深：
  `backend/src/api/routes/payments/index.routes.ts:2-3,7-8` 把 `/stripe`、`/razorpay` 兩個子 router 掛在
  `paymentsRouter` 下（`router.use('/stripe', stripeRouter)` / `router.use('/razorpay', razorpayRouter)`），
  子目錄各自有 `catalog.routes.ts`、`config.routes.ts` 等按功能切分的檔案。
  對應 service 層 `backend/src/services/payments/stripe/`、`backend/src/services/payments/razorpay/`
  各自有 `checkout.service.ts`、`webhook.service.ts`、`subscription.service.ts` 等（stripe 9 個檔、razorpay 7 個檔），
  跨 provider 共用邏輯抽到 `backend/src/services/payments/{helpers.ts, transaction.service.ts, payment-customer.service.ts,
  webhook-store.service.ts, payments-advisory-lock.ts}`。
- **在 `server.ts` 手動註冊，沒有自動掃描/自動註冊機制**：`backend/src/server.ts:8-25` 22 行 import 語句，
  `backend/src/server.ts:205-224` 22 行 `apiRouter.use('/xxx', xxxRouter)`。新增 domain 必須手動加這兩行 —
  這是唯一「觸碰 server.ts」的步驟，其餘都在新目錄裡完成。

### 1.2 是否有一致的 scaffolding 慣例？

- 有清楚的**目錄命名慣例**（routes/services/providers 三層同名子目錄）與**單例模式**
  （所有 service 都用 `static getInstance()`，例如 `FunctionService.getInstance()`
  `backend/src/services/functions/function.service.ts:34`），但**沒有 CLI/generator 腳本**可自動產生骨架
  （未在 `package.json` scripts 或 `backend/scripts/` 找到 `generate:module` 之類工具，⚠️ 未驗證是否有遺漏）。
- 錯誤處理慣例一致：所有 route handler 用 `try { ... } catch (err) { next(err) }`，交給
  `backend/src/api/middlewares/error.ts:8` 的 `errorMiddleware`（`app.use(errorMiddleware)` 在
  `backend/src/server.ts:318`，必須最後掛載）統一輸出格式。
- 認證慣例一致：middleware 從 `@/api/middlewares/auth.js` import（`verifyAdmin`、`verifyUser`、`verifyApiKey` 等），
  逐條 route 疊加，而非整個 router 全域套用 —— 見下方第 5 節。

結論：這是**約定優於配置（convention over configuration）**式的擴充點，而非插件系統；新增領域＝複製既有目錄結構＋在
`server.ts` 加兩行 import/mount。沒有 manifest 檔或動態載入。

---

## 2. Payment provider 擴充點（`backend/src/providers/payments/`）

- **沒有共用的 `interface`/抽象基底**（⚠️ 與 OAuth 的作法明顯不同）。
  `StripeProvider`（`backend/src/providers/payments/stripe.provider.ts:90`）和
  `RazorpayProvider`（`backend/src/providers/payments/razorpay.provider.ts:303`）是各自獨立的 class，
  各自定義自己的 error 型別（`StripeKeyValidationError` at `stripe.provider.ts:61`、
  `RazorpayKeyValidationError` at `razorpay.provider.ts:7`）與各自的資料型別（`RazorpayOrder`、`RazorpayPlan`…
  `razorpay.provider.ts:45-266`），互不繼承。
- 統一的抽象只存在於**型別層**：`backend/src/types/payments.ts:9` 定義
  `export type PaymentProvider = 'stripe' | 'razorpay'`，作為 discriminant 使用在
  `types/payments.ts:251,338,356` 等多個介面的 `provider` 欄位上，資料庫紀錄靠這個欄位分流查詢，
  而非透過多型呼叫共同介面方法。
- 路由層是「並列掛載」而非「策略切換」：`backend/src/api/routes/payments/index.routes.ts:7-8` 直接把
  `/stripe`、`/razorpay` 兩條路由並排掛上，前端/使用者顯式選擇要打哪個 provider 的 endpoint，
  後端不會依專案設定動態選 provider class。
- 因此**新增第三個 payment provider（例如 PayPal）的作法**推測為：
  1. 在 `backend/src/providers/payments/` 新增 `paypal.provider.ts`（仿 stripe/razorpay 的 class 寫法，非 interface 約束）。
  2. 在 `backend/src/services/payments/paypal/` 新增對應 service 子目錄。
  3. 在 `backend/src/api/routes/payments/paypal/` 新增 route 子目錄，於
     `payments/index.routes.ts` 加一行 `router.use('/paypal', paypalRouter)`。
  4. 把 `'paypal'` 加進 `PaymentProvider` union type（`types/payments.ts:9`）。
  ⚠️ 未驗證：因為只有兩個既有實作且無強制 interface，這個「新增第三個」的路徑是根據既有兩者的對稱性推斷，
  沒有官方文件或 abstract class 佐證，屬於**弱擴充點**（可行但無編譯期保障一致性）。

---

## 3. OAuth provider 擴充點（`backend/src/providers/oauth/`）——真正的 Strategy Pattern

與 payments 相反，這裡有**明確定義的 interface**，是本專案最乾淨的 plugin point 之一：

- `backend/src/providers/oauth/base.provider.ts:7-29` 定義 `OAuthProvider` interface：
  - `generateOAuthUrl(state?, additionalParams?): Promise<string>`（必要）
  - `handleCallback(payload: {code?, token?}): Promise<OAuthUserData>`（必要）
  - `handleSharedCallback?(payloadData): OAuthUserData`（可選，共用金鑰情境）
- 7 個既有實作全部落在同一個檔案模式：`google.provider.ts`、`github.provider.ts`、`discord.provider.ts`、
  `linkedin.provider.ts`、`facebook.provider.ts`、`microsoft.provider.ts`、`x.provider.ts`、`apple.provider.ts`、
  外加 `custom.provider.ts`（讓使用者接自訂 OIDC provider，本身也是這套 interface 的一種實作，等於「使用者可設定的擴充點」）。
- `backend/src/providers/oauth/index.ts:1-8` 是 barrel export，把全部 provider class 匯出。
- 新增一個 OAuth provider 的步驟（依此 interface 推斷，✅ 高信心）：
  1. 新建 `backend/src/providers/oauth/<vendor>.provider.ts`，implements `OAuthProvider`。
  2. 在 `backend/src/providers/oauth/index.ts` 加一行 export。
  3. 在 auth service 的 provider 選擇邏輯（⚠️ 未逐行核對，推測位於
     `backend/src/services/auth/` 底下某個 factory/switch，依 provider 名稱 new 對應 class）中登記。
  4. 在 dashboard `features/auth` 的 UI 選單加入按鈕（見第 6 節 `packages/dashboard/src/features/auth`）。
- 這是唯一同時具備「interface 約束」＋「barrel export 匯總」的 provider 擴充點，
  可作為其他模組（payments）未來改善一致性時的範本。

---

## 4. Edge Functions —— 面向終端使用者的主要擴充機制（`functions/`）

這是 InsForge 平台**設計給最終使用者**（不是 InsForge 開發者）新增功能的地方，等同於 Supabase Edge Functions /
Vercel Functions 的角色。目錄下只有 4 項：`deno.json`、`examples/`、`server.ts`、`worker-template.js`。

### 4.1 Function 合約（contract）

- 使用者程式碼必須遵守固定形狀：`module.exports = async function (request) { ... return new Response(...) }`，
  範例見 `functions/examples/demo-hello-world.js:14-61`（CommonJS `module.exports`、標準 Fetch API 的
  `Request`/`Response`、`Deno.env.get()` 讀 secrets）。另有 `functions/examples/demo-whoami.js` 示範
  `createClient`（InsForge SDK client，由 runtime 注入）。`functions/examples/README.md` 有更多說明。
- `functions/worker-template.js` 是**執行殼**，每個 request 建立一個全新的 Web Worker（一次性、用完即銷毀），
  在把使用者程式碼跑起來之前做了大量安全加固：
  - `worker-template.js:14-51`：Top-level 就 shadow 掉 `Deno.env`（凍結成唯讀的 sterile 版本，
    `set()/delete()` 會 throw）與 `process.env`，防止使用者程式碼讀取真正的環境變數／逃逸沙箱
    （註解稱為 "SECURITY BLACKOUT"）。
  - `worker-template.js:63-75`：在任何 top-level await 之前先同步註冊 `self.onmessage` 緩衝早到訊息，
    避免動態 import 期間訊息遺失導致 504（有詳細註解說明這是修過的 race condition）。
- `functions/server.ts:250` 的 `Deno.serve({ hostname, port }, handler)` 是本地 Deno runtime 入口，
  負責讀 `worker-template.js`（`server.ts:22-29` `getWorkerTemplateCode()`）、解密 secrets
  （`server.ts:32-` `decryptSecret`，AES-GCM，與 Node 端加密相容）、建立 Worker 並丟訊息進去執行。

### 4.2 部署與呼叫路徑（backend 這一側）

- `backend/src/services/functions/function.service.ts`（846 行）是 orchestrator：singleton
  （`getInstance()` at line 34），部署呼叫 `denoSubhostingProvider.deployFunctions(...)`
  （`function.service.ts:449`, `:553`），並有 `isSubhostingConfigured()` / `getDeploymentUrl()` /
  `syncDeployment()` 等方法供 `server.ts` 開機時同步（`backend/src/server.ts:343-348`）。
- `backend/src/providers/functions/deno-subhosting.provider.ts`（1160 行）封裝 Deno Deploy 的 Subhosting API
  （`export class DenoSubhostingProvider` at line 290），是雲端部署的 provider 實作；本地開發則直接打
  `functions/server.ts` 起的本地 Deno runtime（`appConfig.functions.denoRuntimeUrl`）。
- 請求路由：`backend/src/server.ts:231-289` 的 `app.all('/functions/:slug', ...)` 是舊版相容代理，
  優先轉發到 Subhosting 部署 URL，否則 fallback 本地 runtime（`server.ts:239-241`）；
  新的呼叫慣例是 SDK 直連 edge function（註解見 `server.ts:230`）。
  另外 `/api/functions` 底下（`backend/src/api/routes/functions/index.routes.ts`，經 `server.ts:14,211` 掛載）
  是管理 API（CRUD functions 本身），與實際「執行」function 的路徑（`/functions/:slug`）是分開的兩條路。

### 4.3 小結

Edge Functions 是**沙箱化、按合約（contract-based）**的擴充點：使用者不需要碰 InsForge 原始碼，
只要寫符合 `module.exports = async function(request)` 形狀的 JS/TS 檔案，透過 dashboard 或 API 部署即可。
這與 backend route/service 的「複製目錄結構」擴充方式性質完全不同（後者是給 InsForge 貢獻者，前者是給終端使用者）。

---

## 5. Middleware chain（`backend/src/api/middlewares/`）

共 5 個檔案：

| 檔案 | 匯出 | 用途 |
|---|---|---|
| `auth.ts` | `verifyUser` (`:93`), `verifyAdmin` (`:112`), `verifyApiKey` (`:163`), `verifyAnonKey` (`:201`), `verifyToken` (`:235`), `verifyCloudBackend` (`:284`), 型別 `AuthRequest`/`UserContext`，helper `extractBearerToken`/`extractApiKey`/`extractAnonKey` | 認證/授權，多種角色對應多種 middleware，逐條 route 疊加使用（例如 `webscraperRouter.get('/apify/connection', verifyAdmin, handler)`，見 `backend/src/api/routes/webscraper/index.routes.ts:11`） |
| `error.ts` | `errorMiddleware` (`:8`) | 全域錯誤處理，`app.use(errorMiddleware)` 掛在最後（`server.ts:318`），必須排在所有 route 之後 |
| `rate-limiters.ts` | `sendEmailOTPRateLimiter` (`:53`), `s3AccessKeyManagementRateLimiter` (`:79`), `computeLogsRateLimiter` (`:106`), `verifyOTPRateLimiter` (`:130`), `perEmailCooldown` (`:154`), `functionsWriteLimiter`/`deploymentsWriteLimiter`/`computeWriteLimiter` (`:448-450`, 由 `createWriteEndpointLimiter('functions'|'deployments'|'compute')` 動態產生)，另有可從資料庫熱更新上限的 `applyWriteEndpointLimits`/`startWriteEndpointLimitsRefresh` (`:287,389`) | 按 endpoint 分類的 rate limiting，`WriteLimiterCategory` type (`:232`) 是擴充新分類時要改的地方 |
| `s3-sigv4.ts` | `s3Sigv4Middleware` (`:56`), 型別 `S3AuthContext`/`S3AuthenticatedRequest` | S3 相容協議的 SigV4 簽章驗證，只掛在 `s3GatewayRouter`（`server.ts:166`，特別注意要在 JSON body parser **之前**掛載以保留原始 stream，見 `server.ts:163-166` 註解） |
| `upload.ts` | `upload`/`dynamicUploadSingle`/`handleUploadError`/`processFormData`, `getMaxFileSize` | multer 檔案上傳封裝 |

### 組裝順序（`backend/src/server.ts`）

全域 middleware 依序掛載（順序敏感，註解都有說明原因）：
1. `cors()` (`server.ts:82-88`)
2. `cookieParser()` (`:89`)
3. 自訂 HTTP logging middleware（inline function，`:90-157`，攔截 `res.send`/`res.json` 算 response size）
4. `/api/webhooks` 用 `express.raw({type:'application/json'})` + `webhooksRouter`（`:161`，必須在 JSON parser **之前**掛，才能保留原始 bytes 做簽章驗證）
5. `/storage/v1/s3` 用 `s3GatewayRouter`（`:166`，同理必須在 JSON parser 之前，走 raw stream）
6. `express.json()` / `express.urlencoded()`（`:175-176`，限制大小可由環境變數覆寫）
7. 22 個 domain router 掛在 `apiRouter`，再 `app.use('/api', apiRouter)`（`:205-227`）
8. 靜態檔案／SPA fallback／404 handler（`:298-316`）
9. `errorMiddleware`（`:318`，最後）

**擴充點**：新增 middleware 的慣例是要嘛整條 route 疊加式使用（像 `auth.ts` 的各種 `verify*`），要嘛全域
`app.use()` 到 `createApp()` 裡——沒有 middleware registry/plugin loader，順序完全靠手動排列且順序敏感
（webhooks/S3 gateway 必須搶在 body parser 前面），改動時需特別小心。

---

## 6. Dashboard feature 模組結構（`packages/dashboard/src/features/`）

- `packages/dashboard/package.json:29-39` 定義了 Node subpath imports（`#xxx` import map），
  是刻意設計的模組邊界：
  ```
  #app/*        -> ./dist/app/*.js
  #assets/*     -> ./src/assets/*
  #components   -> ./dist/components/index.js   (barrel)
  #components/* -> ./dist/components/*.js
  #features/*   -> ./dist/features/*.js
  #layout/*     -> ./dist/layout/*.js
  #lib/*        -> ./dist/lib/*.js
  #navigation/* -> ./dist/navigation/*.js
  #router/*     -> ./dist/router/*.js
  #types(/*)    -> ./dist/types/...
  ```
  這代表官方鼓勵 feature 之間、feature 與 app-shell 之間走**絕對路徑 import**（`#features/payments/...`），
  不要用深層相對路徑，是「模組邊界」而非嚴格 plugin isolation（沒有 runtime 隔離，只是 import 慣例＋
  build 出 dist 後才生效，會強制檔案需先 build 才能被其他 package 用）。
- `packages/dashboard/src/features/` 目前有 14 個 feature 目錄：
  `ai, analytics, auth, compute, dashboard, database, deployments, functions, login, logs, payments,
  realtime, storage, visualizer, webscraper`。
- 一致的內部結構（以 `compute`、`ai`、`database`、`auth`、`realtime`、`logs` 為樣本）：
  ```
  features/<name>/components/   -- React 元件
  features/<name>/hooks/        -- custom hooks（資料抓取、狀態）
  features/<name>/services/     -- 呼叫 backend API 的 client 層
  features/<name>/pages/        -- 路由對應的頁面元件
  features/<name>/lib/          -- (可選) 工具函式，例如 compute
  features/<name>/contexts/     -- (可選) React context，例如 database
  features/<name>/templates/    -- (可選)，例如 database
  features/<name>/__tests__/    -- (可選)
  ```
  這與 backend 的 routes/services/providers 三層慣例是類似的「約定優於配置」，但同樣**沒有 generator 腳本**
  （⚠️ 未驗證是否有 plop/hygen 之類工具，掃描 `packages/dashboard` 未發現）。
- 側邊選單註冊：`packages/dashboard/src/navigation/menuItems.ts` 是**手動陣列**
  （`dashboardStaticMenuItems: DashboardPrimaryMenuItem[]`，`menuItems.ts:40` 起），
  每個 feature 若要出現在 dashboard 側欄，需要手動在這個陣列加一筆（含 `id/label/href/icon`，
  `icon` 從 `lucide-react` import，見 `menuItems.ts:1-21`）。同 backend `server.ts` 的手動註冊模式一致：
  **新增 dashboard feature = 建目錄 + 手動改一個中央清單檔**，不是自動探索。

---

## 7. UI component library 擴充（`packages/ui/src/`）

- 目錄結構：`packages/ui/src/{components, lib, styles.css, test}`，`components/` 底下每個元件一個檔案
  （例如 `Button.tsx`, `DropdownMenu.tsx`, `Select.tsx` ... 依 barrel 內容推斷）。
- **兩層 barrel export**：
  - `packages/ui/src/components/index.ts`（97 行）— 每個元件目錄內部彙整，例如
    `export { Button, buttonVariants, type ButtonProps } from './Button';`（`components/index.ts:1`）。
  - `packages/ui/src/index.ts`（103 行）— package 對外的公開 API，統一 re-export 自 `./components`
    （例如 `export { Button, buttonVariants, type ButtonProps } from './components';`，`src/index.ts:2`），
    並額外匯出 `cn` utility（`src/index.ts:102`，來自 `./lib`）。
- **新增一個 design-system 元件的步驟**（依既有慣例推斷，✅ 高信心）：
  1. 在 `packages/ui/src/components/` 新增 `NewThing.tsx`。
  2. 在 `packages/ui/src/components/index.ts` 加一行 `export { NewThing, ... } from './NewThing'`。
  3. 在 `packages/ui/src/index.ts` 加一行對應的 re-export（讓外部 package 能 `import { NewThing } from '@insforge/ui'` ⚠️ 套件名稱未逐一核對 package.json）。
  4. 若元件有 variant，慣例是用 `class-variance-authority` 風格的 `xxxVariants` 一併 export
     （模式可見於 `buttonVariants`、`switchVariants`、`badgeVariants`、`toastVariants`）。
- Tailwind 擴充點：`packages/ui/tailwind-preset.js` 是共用的 Tailwind preset，供 dashboard 等消費端
  `presets: [require('@insforge/ui/tailwind-preset')]`（⚠️ 未逐行核對 preset 內容與消費端設定檔）。
- 沒有發現 Storybook 或元件 registry/manifest 檔案（⚠️ 未全面搜尋，僅檢查了 `packages/ui/src` 頂層）。

---

## 8. 整體模式歸納

| 擴充點 | 機制型態 | 是否有 interface/type 約束 | 是否有 registry 檔要手動改 | 面向對象 |
|---|---|---|---|---|
| Backend route/service/provider | 目錄慣例 + singleton | 否（無共用 interface） | 是（`server.ts` import + mount） | InsForge 貢獻者 |
| Payment provider | 並列 class，無抽象基底 | 否 | 是（`payments/index.routes.ts` + `PaymentProvider` union type） | InsForge 貢獻者 |
| OAuth provider | Strategy pattern | **是**（`OAuthProvider` interface） | 是（`oauth/index.ts` barrel + auth service 內的 factory，⚠️ factory 位置未逐行核對） | InsForge 貢獻者 |
| Edge Functions | Contract-based 沙箱執行 | 是（固定 `module.exports = async function(request)` 形狀） | 否（動態部署，不需改 core 程式碼） | **終端使用者**（平台真正的 plugin 系統） |
| Middleware | 手動疊加 / `app.use()` | 部分（`AuthRequest`/`S3AuthenticatedRequest` 型別） | 是（`server.ts` 內順序敏感排列） | InsForge 貢獻者 |
| Dashboard feature | 目錄慣例 + `#imports` map | 否 | 是（`navigation/menuItems.ts` 手動陣列） | InsForge 貢獻者 |
| UI component | 兩層 barrel export | 部分（各元件自己的 Props type） | 是（`components/index.ts` + `src/index.ts` 各加一行） | InsForge 貢獻者 |

**核心觀察**：InsForge 內部的擴充機制普遍是「約定優於配置＋手動登記到一個中央清單檔」，
沒有自動發現/動態載入的 plugin loader；唯二例外是 (a) OAuth provider 有正式 interface 可 implements，
(b) Edge Functions 是唯一真正意義上「不需碰 InsForge 原始碼」的使用者側擴充系統，其餘都要求
fork/clone 這個 monorepo 才能新增功能。
