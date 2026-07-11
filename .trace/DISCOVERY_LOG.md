# Discovery Log — 探索紀錄與待解問題

> Trace 產出，來源 commit: `main`@`0dd55c5`（見 `.trace/TRACE_META.md`）。本文件彙整 `.trace/_context/*.md` 六份工作文件與 web search 的發現，聚焦「文件 vs 程式碼落差」「TODO/技術債」「未解問題」。與其餘 `_context/*.md` 一樣，本文件本身也遵守 ⚠️ 未驗證 標注慣例。

---

## 1. Web Search 發現摘要

完整內容見 `.trace/_context/web_findings.md`，此處濃縮重點：

- [GitHub - InsForge/InsForge](https://github.com/InsForge/insforge)：官方 repo，定位「the all-in-one, open-source backend platform for agentic coding」，Apache-2.0。
- [InsForge Docs](https://docs.insforge.dev/introduction) / [insforge.dev](https://insforge.dev/)：官方文件與官網。
- [InsForge 2.0 Launch 部落格](https://insforge.dev/blog/insforge-launch-v2)：2.0 版公告，資料庫建立數成長 500%，近 99% 操作來自 agent 而非人類，provisioning/config/migration/runtime 皆由 agent 完成。
- [GIGAZINE 報導](https://gigazine.net/gsc_news/en/20260614-insforge/)：把 InsForge 類比為「給 AI coding agent 用的 Heroku」。
- [DeepWiki: InsForge/InsForge](https://deepwiki.com/InsForge/InsForge)：第三方自動生成 wiki，可作交叉驗證，但非官方、⚠️ 未逐頁核對。
- Y Combinator P26 batch 孵化、比 Supabase 快 1.6 倍／省 2.4 倍 token 的宣稱：⚠️ 未驗證（皆為第三方報導/測評，無官方 benchmark 方法論佐證）。
- 核心 building blocks（Postgres+pgvector、S3 相容儲存、OAuth、Deno edge functions、OpenAI 相容 model gateway、realtime sync）與本次程式碼掃描結果一致，屬於少數已交叉驗證的市場定位資訊。
- 未執行方向：GitHub Discussions/Issues 中的架構討論串、官方 Discord、與 Supabase/Firebase 的完整技術比較文章 — 皆因大型 monorepo 全套件 trace 的時間考量而略過。

---

## 2. 既有文件與程式碼之間的落差清單

| # | 文件 | 文件描述 | 程式碼實際 | 位置 | 嚴重度 |
|---|---|---|---|---|---|
| 1 | `CONTRIBUTING.md:24` | 「`/backend` - Core backend service with Express.js, PostgreSQL, and **Better Auth** integration」 | 完全找不到 `better-auth` 套件依賴或任何 import；backend 的認證是**自研**的 JWT/PKCE 系統：`TokenManager`（RS256/HS256 雙演算法，`backend/src/infra/security/token.manager.ts:46-65`）、`OAuthPKCEService`（`backend/src/services/auth/oauth-pkce.service.ts:26-42`），底層用 `jose`/`jsonwebtoken`/`bcryptjs`，並非 Better Auth 這個第三方函式庫 | `backend/package.json`（無 `better-auth` dependency）、`backend/src/infra/security/token.manager.ts`、`backend/src/services/auth/oauth-pkce.service.ts` | **高**——可能誤導新貢獻者去找一個不存在的整合點，或以為認證邏輯可以直接參考 Better Auth 官方文件 |
| 2 | `CONTRIBUTING.md:20-29`（Project Structure） | 只列出 `/backend`、`/frontend`、`/packages/shared-schemas`、`/docs`、`/functions`、`docker-compose.yml` | 完全未提及 `packages/ui`（設計系統元件庫）與 `packages/dashboard`（真正的 dashboard 功能實作，`frontend/` 只是 self-hosting 用的殼層掛載點） | `packages/ui/src/index.ts`、`packages/dashboard/src/index.ts`、`frontend/src/App.tsx:1-13` | **高**——新貢獻者要修 dashboard UI 邏輯很可能先跑去改 `frontend/`，改完發現沒作用（因為真正邏輯在 `packages/dashboard/`） |
| 3 | `README.md:97` | 「**Compute** (private preview): Long-running container services」 | `backend/src/services/compute/`、`backend/src/providers/compute/fly.provider.ts`（1000+ 行完整實作，含 Fly.io Machines API、IP 分配、狀態輪詢重試）、`backend/src/api/routes/compute/`、`packages/dashboard/src/features/compute/`（含 lib/ 子目錄）均已是**功能完整**的實作，非早期原型；docker-compose 中已有 `FLY_API_TOKEN`/`FLY_ORG` 環境變數與 self-host/cloud 雙模式判斷（`services.service.ts:250-276`） | `backend/src/providers/compute/fly.provider.ts:55-`、`backend/src/services/compute/services.service.ts:250-276` | **中**——"private preview" 標籤可能只反映商業/支援政策（例如尚未正式 SLA），而非程式碼成熟度，需向維護者確認 |
| 4 | `CONTRIBUTING.md:27` | 「`/docs` - **MCP documentation**」 | `docs/` 實際涵蓋範圍遠大於 MCP：`core-concepts/`（ai/analytics/authentication/compute/database/functions/messaging/payments/realtime/sites/storage）、多語言 `sdks/`（typescript/kotlin/swift/rest）、`agent-native/`、`examples/`、`showcase/`、`superpowers/`，是完整的 Mintlify 產品文件站，MCP 只是其中一小部分 | `docs/docs.json`、`docs/core-concepts/*`、`docs/sdks/*` | **低**——描述過窄，但不算錯誤，只是資訊不足 |
| 5 | `README.md`（quickstart／一鍵部署） | 列出 Railway/Zeabur/Sealos 一鍵部署 | `deploy/` 目錄下另有 `docker-compose.dokploy.yml`、`zeabur` 設定；`.agents/docs/deployment.md` 另有 agent 專用部署補充說明，README 未連結或提及這份文件 | `deploy/docker-compose.dokploy.yml`、`.agents/docs/deployment.md` | **低**——非錯誤，屬資訊分散（同一主題文件散落在 README / `.agents/docs/` 兩處，未互相連結） |
| 6 | `README.md` 圖示（Mermaid 架構圖） | 圖中列出的 core products 未明確標示「PostgREST 與 Express backend 是兩條並存的資料存取路徑」這個關鍵架構事實 | 實際上 record CRUD 走 PostgREST proxy、table DDL 與 dashboard record 操作走 Express 直連 `pg`，兩者用 `NOTIFY pgrst, 'reload schema'` 同步（見 `data_flow.md`） | `backend/src/api/routes/database/records.routes.ts`、`backend/src/api/routes/database/admin.routes.ts`、`backend/src/services/database/database-table.service.ts:266-271` | **中**——這是全專案最獨特的機制之一，但對外文件（README/docs）未特別強調，可能造成 SDK 使用者對「為什麼 dashboard 看到的資料範圍跟我的 API key 查到的不一樣」感到困惑 |
| 7 | `packages/ui` 對外沿用 | `packages/ui/src/index.ts` barrel export 有 20+ 元件，但未見任何 Storybook 或 component registry/manifest（`extensions.md` 第 7 節已確認） | 對外使用者若想瀏覽可用元件只能讀原始碼 | `packages/ui/src/index.ts`、`packages/ui/src/components/index.ts` | **低**——非文件錯誤，屬於文件缺口（沒有元件目錄可查） |

**落差數量統計**：本次共發現 **7 項**文件-程式碼落差，其中 高 2 項、中 2 項、低 3 項。落差 #1（Better Auth）與 #2（package boundary）建議優先修正，因為會直接誤導新貢獻者的開發路徑。

---

## 3. 程式碼中的 TODO / FIXME / HACK 彙整

**執行指令**：
```
grep -rn "TODO\|FIXME\|HACK\|XXX" backend/src packages/dashboard/src packages/ui/src packages/shared-schemas/src functions --include="*.ts" --include="*.tsx" --include="*.js"
```

**結果：0 筆命中。** 進一步放寬搜尋範圍（不限副檔名、含 `frontend/`、大小寫不敏感的 `todo`）後，`backend/tests/integration/rls.test.ts` 內出現的 16 筆 `todo` 全部是**測試資料表名稱**（RLS 測試建立一張叫 `todos` 的示範表，`INSERT INTO todos ...`），與程式碼標記語意無關。

**觀察**：InsForge 原始碼（`backend/src/`、`packages/dashboard/src/`、`packages/ui/src/`、`functions/`）中**完全沒有** `TODO`/`FIXME`/`HACK`/`XXX` 風格的程式碼標記註解。可能原因（⚠️ 未驗證，皆為推測）：

1. Issue-first 開發流程（`CONTRIBUTING.md`：需先 claim issue，每人最多 3 個 open assigned issues）把「未完成事項」都外部化到 GitHub Issues，而非留在程式碼註解裡。
2. PR 前必須通過 `npx turbo run lint`（`.claude/skills/insforge-dev` 提及的檢查清單），若 lint 規則含 `no-warning-comments` 之類規則會阻擋 TODO 註解合併，⚠️ 但本次未在 `eslint.config.*`/`.eslintrc*` 中找到此規則的顯式設定，需再確認。
3 repo 歷史可能經過 squash/rebase，移除了開發過程中的暫時性標記。

這代表本節在本次 trace 中沒有「TODO 清單」可彙整，改為記錄「codebase 異常乾淨」這個發現本身，並建議向維護者確認是否有內部（非程式碼內）的技術債追蹤機制（例如 `.internal/docs/audits/` 或 `.internal/docs/plans/`，本次 trace 依 `recon.md` 5.5 節的既定原則未深入讀取其內容）。

---

## 4. 未解答的疑問或模糊地帶（⚠️ 未驗證彙整）

`Grep "⚠️ 未驗證" .trace/_context/*.md` 共命中 **30 筆**，分佈：core_logic.md 4、entry_points.md 6、extensions.md 3、integrations.md 7、web_findings.md 2、data_flow.md 5、configuration.md 3。依主題歸類如下：

### 4.1 啟動/初始化流程
- `backend/src/utils/seed.ts` 完整 seed 邏輯（是否建立預設 admin、預設 auth provider 設定等）未逐行讀取，只確認呼叫鏈（`entry_points.md` 1.4）。
- `TelemetryService` 是否在 `getInstance()` 內部 lazy-construct，或要求先手動 `new`（`entry_points.md` 1.6，見第 5 節技術債）。
- `functions/worker-template.js` 完整內容：message buffering 之後如何實際 `eval`/`import` 使用者 function code、如何組裝 `Request`/`Response`、如何把結果 postMessage 回主 thread（`entry_points.md` 5.2）。
- `functions/deno.json` 內容（`entry_points.md` 5.3）。
- `backend/scripts/check-migration-duplicates.js`、`test-deno-subhosting.sh`、root `scripts/*.sh` 的具體實作（`entry_points.md` 6.1/6.2）。
- `baseline-migrations.js` 完整邏輯（僅掃到函式簽名 `readMigrationNames`/`baselineMigrations`）（`entry_points.md` 6.3、`configuration.md` 8）。
- Docker Compose 各檔案（`docker-compose.yml`/`.override.yml`/`.prod.yml`/`.dokploy.yml`）間的具體 override 關係與各自 command（`entry_points.md` 7）。
- `frontend/src/helpers.ts` 的 `isInIframe()` 用於何處未驗證（`entry_points.md` 2.2）。
- `frontend/src/cloud-hosting/` 完整實作（partner iframe 通訊協定等）（`entry_points.md` 2.4、8）。
- `packages/dashboard/src/router/AppRoutes.tsx`、`RequireAuth.tsx` 完整路由表與登入保護邏輯（`entry_points.md` 3.2、8）。

### 4.2 Database / Data Flow
- `backend/src/api/app.ts`（或等價掛載點）內 `/api/database` 確切頂層掛載路徑與 middleware 順序未直接讀取確認（`data_flow.md`）。
- database migrations runner 是否為 `node-pg-migrate` 套件本身或自訂 runner，未逐一核對 `package.json` 相依（`data_flow.md`；⚠️ 注：`configuration.md` 8 節已確認 `migrate:up` 使用 `node-pg-migrate up`，此項可能已部分澄清，建議收斂）。
- `system.update_updated_at()` trigger function 的定義來源 migration 檔未定位（`data_flow.md`、`core_logic.md` 1.2）。
- `AdminRecordService`（`admin-record.service.ts`）SQL 實作細節未逐行閱讀（`data_flow.md`）。
- Dashboard 不走 PostgREST proxy 而走 admin 直連的確切設計原因（RLS bypass？功能需求？）程式碼內無註解佐證，屬推測（`data_flow.md`）。
- `ApiClient` 的 401 自動 refresh + retry 完整邏輯未逐行讀取（`data_flow.md`）。
- `database-migration.service.ts` 完整內容未讀，與 `database-table.service.ts` 的 DDL 邏輯是否重複/分工未確認（`core_logic.md` 7）。

### 4.3 Auth / Security
- `checkSqlExecutionGuards` 是否覆蓋所有可能的提權路徑（例如透過 extension function 間接改變 session 狀態）（`core_logic.md` 2）。
- OAuthPKCEService 用 in-memory `Map` 存 PKCE code，多實例部署時是否有 sticky routing 或改用 Redis/DB 的計畫未驗證（`core_logic.md` 7；亦見第 5 節技術債）。
- AI Gateway 的 `embedding.service.ts`/`image-generation.service.ts` 是否也都只走 `OpenRouterProvider.sendRequest()` 同一條路徑，未逐一確認（`core_logic.md` 3.3、7）。
- `isOAuthSharedKeysAvailable()` 實際呼叫端與行為未追蹤（`configuration.md` 5、待補充清單）。
- `FunctionService.isSubhostingConfigured()` 判斷式細節未讀（`configuration.md` 3、5、待補充清單）。
- `backend/src/utils/trust-proxy.ts` 的 `parseTrustProxySetting` 邏輯未細看（`configuration.md` 1、待補充清單）。
- Postgres container 的 `app.encryption_key` GUC 與 Node 端 `EncryptionManager` 是否共用/如何協作，是否牽涉 pgcrypto column-level encryption（`configuration.md` 4）。
- `database/base.provider.ts` 或 provider factory 如何依環境變數選用 `CloudDatabaseProvider` vs 本地 provider（`configuration.md` 5、待補充清單）。
- `frontend/` 目錄如何把 `VITE_API_BASE_URL` 傳遞成 `InsforgeDashboard` 的 `host.backendUrl` prop，完整鏈路未追完（`configuration.md` 6、待補充清單）。

### 4.4 外部整合（Integrations）
- Stripe/Razorpay SDK 是否使用其內建 `maxNetworkRetries`/自動重試，未見顯式覆寫（`integrations.md` 待確認 1）。
- OAuth providers 的 axios 呼叫是否設有逾時；抽樣片段中未見 `timeout` 參數（`integrations.md` 待確認 2）。
- `SmtpEmailProvider.send()` 完整方法體（僅讀方法簽章與註解）（`integrations.md` 待確認 3）。
- `webscraper/apify.provider.ts` 除 `getConnection`/token 相關方法外，其餘方法（runs/actors/datasets 查詢）的錯誤處理與 retry 細節未核對（`integrations.md` 待確認 4）。
- `compute/cloud.provider.ts`、`compute.provider.ts`（cloud-managed compute provider）未逐行核對（`integrations.md` 待確認 5）。
- `CW_SUFFIX_*` 系列環境變數是否記錄於使用者可見文件（`.env.example` 未見）（`integrations.md` 待確認 6）。
- Facebook OAuth 對應的 `FACEBOOK_CLIENT_ID`/`FACEBOOK_CLIENT_SECRET` 未見於 `.env.example`，此功能是否對外可用尚待確認（`integrations.md` 待確認 7）。

### 4.5 其他
- `packages/ui/tailwind-preset.js` 具體被 `frontend/`、`packages/dashboard/` 的 `tailwind.config` 消費端如何引用，未逐行核對（`entry_points.md` 4.2）。
- OAuth provider 新增流程中「auth service 的 provider 選擇邏輯（factory/switch）」位置未逐行核對（`extensions.md` 3）。
- DeepWiki（第三方自動生成 wiki）內容未逐頁核對，僅列為可能的交叉驗證來源（`web_findings.md`）。
- Y Combinator 孵化資訊、Supabase 效能對比數據，皆僅來自第三方報導（`web_findings.md`）。

---

## 5. 已知技術債（架構層面觀察）

1. **Payments provider 沒有共用 interface**（`extensions.md` 第 2 節）：`StripeProvider`（`backend/src/providers/payments/stripe.provider.ts:90`）與 `RazorpayProvider`（`backend/src/providers/payments/razorpay.provider.ts:303`）各自獨立 class，各自定義錯誤型別與資料型別，互不繼承；只有型別層的 `PaymentProvider = 'stripe' | 'razorpay'` discriminant（`backend/src/types/payments.ts:9`）做鬆散串接。相較之下 OAuth 有明確的 `OAuthProvider` interface（`backend/src/providers/oauth/base.provider.ts:7-29`）。新增第三個 payment provider（例如 PayPal）沒有編譯期保障一致性，屬於「弱擴充點」。

2. **TelemetryService 的 constructor 模式與其他 singleton 不一致**（`entry_points.md` 1.6）：專案內幾乎所有 service/manager 都是 `private constructor` + `static getInstance()`，唯獨 `TelemetryService`（`backend/src/services/telemetry/telemetry.service.ts:104-114`）是 `public constructor` 但仍提供 `getInstance()`，同時允許直接 `new` 與 singleton 存取兩種模式並存，容易造成誤用（例如意外建立第二個實例導致 telemetry 狀態不一致）。

3. **OAuthPKCEService 用 process-local in-memory `Map` 存 PKCE code**（`core_logic.md` 4.2）：多實例水平擴展部署時，簽發 PKCE code 的實例與接手 OAuth callback 的實例若不是同一台，交換會失敗；未見 sticky routing 或改用 Redis/DB 的替代設計證據。這是雲端多實例部署下的潛在正確性風險。

4. **AI Gateway 的 provider 抽象層不完整**（`core_logic.md` 3.3、6.2）：`backend/src/providers/ai/` 只有 `OpenRouterProvider` 一個 concrete provider，沒有共用 interface，與其餘子系統（OAuth/Storage/Email/Logs/Compute）的 `base.provider.ts` + N 個 `*.provider.ts` 模式不一致。目前靠 OpenRouter 自身聚合多 LLM，若未來要繞過 OpenRouter 直連某 LLM vendor，需要先補上抽象層。

5. **無自動 scaffolding／plugin loader，全靠手動登記到中央清單檔**（`extensions.md` 第 8 節總結）：新增 backend domain 需手動改 `server.ts`（22 行 import + 22 行 mount，無自動掃描）；新增 dashboard feature 需手動改 `navigation/menuItems.ts`；新增 UI 元件需手動改兩層 barrel export。這不是 bug，但代表擴充成本會隨模組數量線性增加，且容易因為忘記手動登記而出現「功能寫好了但沒被掛載」的疏漏（本次 trace 未發現具體案例，屬於流程風險而非已知 bug）。

6. **Retry / Circuit breaker 策略不一致**（`integrations.md` 第 11 節）：Vercel、Deno Deploy 有完整的 429 backoff + jitter 重試；Fly.io 只有輪詢式 retry（非 error-triggered backoff）；OpenRouter、Stripe/Razorpay、OAuth providers、Storage(S3)、Email(SMTP) 幾乎沒有自動 retry，失敗即拋錯給呼叫端。整個 `backend/src/providers/` 沒有任何 circuit breaker 實作（如 `opossum`）。這代表外部服務抖動時的韌性程度因 provider 而異，缺乏統一策略。

7. **Config 沒有 runtime schema validation**（`configuration.md` 第 1 節）：`app.config.ts` 的 `loadConfig()` 全部用 `process.env.XXX || <default>` 手寫組裝，型別靠 TS interface 而非 zod 等 runtime validation library；即使專案其餘部分（`shared-schemas`）大量使用 Zod 做 request/response 驗證，config 層卻是例外，環境變數格式錯誤要等到實際使用時才會爆炸而非啟動時 fail-fast。

8. **dotenv 三層重複載入**（`configuration.md` 第 2 節）：`app.config.ts`、`server.ts`、`dotenv-cli`（`npm run dev`）各自獨立呼叫 dotenv 載入邏輯，雖然因為 dotenv 預設不覆寫已存在變數而不會產生錯誤結果，但屬於可簡化的重複程式碼，且三層路徑搜尋邏輯不完全一致（4 條候選路徑 vs 1 條候選路徑）增加了理解成本。

9. **`isCloudEnvironment()` 與前端 `isInsForgeCloudProject` 判斷邏輯不同步**（`configuration.md` 第 5-6 節）：後端用 `AWS_INSTANCE_PROFILE_NAME` 是否設定判斷 cloud 環境；前端用 `hostname.endsWith('.insforge.app')` 判斷。兩套邏輯位置分散、依據不同，⚠️ 未驗證是否存在兩者判斷結果不一致的邊界情境（例如自訂網域的 cloud 部署）。

---

## 6. 需要更深入調查的區域

以下模組本次 trace 因時間/深度考量未展開，依優先度排列（優先度依「使用者影響面」與「安全敏感度」評估，⚠️ 屬主觀判斷）：

1. **`functions/worker-template.js` 的完整 message buffering 與程式碼執行機制**——沙箱安全的核心，只確認了開頭的 security blackout 與訊息緩衝的「為什麼」，沒有確認「怎麼做」。安全敏感度高。
2. **`backend/src/utils/seed.ts` 完整邏輯**——初次部署時的預設帳號/設定行為，直接影響 self-host 使用者的初始安全狀態。
3. **`backend/src/infra/database/migrations/bootstrap/baseline-migrations.js`**——資料庫還原/分支修復流程的關鍵腳本，若邏輯有誤可能導致 migration ledger 與實際 schema 不一致。
4. **`frontend/src/cloud-hosting/`（`CloudHostingDashboard.tsx`、`useCloudHosting.ts`、`partner.service.ts`）**——cloud 版特有的 partner iframe 通訊協定，完全未展開。
5. **`packages/dashboard/src/router/RequireAuth.tsx`**——登入保護路由邏輯，屬於前端安全邊界的一部分。
6. **`backend/src/services/database/database-migration.service.ts`**——與 `database-table.service.ts` 的關係/分工未釐清，可能有重複邏輯。
7. **Payment providers（Stripe/Razorpay）的 SDK 內建 retry 行為**與 **OAuth providers 的逾時設定**——關係到外部服務異常時的系統韌性，目前僅抽樣讀取。
8. **`backend/src/providers/database/cloud.provider.ts` 與 provider 選用 factory**——決定 cloud-proxied self-host 模式如何切換資料庫來源，屬於部署拓樸的關鍵分支點。
9. **`.internal/docs/audits/`、`.internal/docs/plans/`、`.internal/docs/specs/`**——本次 trace 依既定原則未讀取內容（非公開/可能是草案），但若要做更完整的技術債盤點，這些文件可能包含維護者已知但未公開的問題清單。

---

## 7. 與維護者或社群確認的問題清單

1. `CONTRIBUTING.md` 提到 backend 用「Better Auth integration」，但程式碼中找不到 `better-auth` 套件或相關 import，實際是自研 JWT/PKCE 系統——這是文件過時（曾經評估過 Better Auth 但後來自研取代）還是筆誤？是否該更新 `CONTRIBUTING.md`？
2. README 把 **Compute** 標示為「private preview」，但程式碼（`backend/src/services/compute/`、`fly.provider.ts`）看起来已是功能完整的實作——這個標籤反映的是程式碼成熟度，還是商業/支援政策（例如尚未提供 SLA 或尚未大規模宣傳）？
3. `CONTRIBUTING.md` 的 Project Structure 段落完全未提及 `packages/ui`、`packages/dashboard` 兩個套件，是否為刻意省略（假設新貢獻者主要只會碰 `backend`）？是否考慮補充，避免新人誤改 `frontend/` 卻發現沒作用？
4. Payments provider（Stripe/Razorpay）刻意不設共用 interface，是設計決策（因為兩家 API 形狀差異太大不值得硬抽象）還是純粹尚未來得及重構？未來若要新增第三個 payment provider，是否有計畫補上抽象層？
5. `OAuthPKCEService` 用 process-local in-memory `Map` 存 PKCE exchange code——InsForge Cloud 若跑多實例，是否有 sticky session routing 保證同一個使用者的 OAuth callback 落在同一台實例？還是這是已知限制、cloud 目前仍是單實例？
6. 整個 `backend/src/providers/` 沒有 circuit breaker，各 provider 的 retry 策略也不一致（Vercel/Deno Deploy 有完整 429 backoff，Fly.io 只有輪詢，OpenRouter/Stripe/OAuth 幾乎無 retry）——這是否為刻意的設計取捨（讓錯誤盡快浮現給呼叫端處理），還是待補的技術債？
7. 為什麼 dashboard 的 record CRUD 操作（`admin.routes.ts`）不透過 PostgREST proxy（`records.routes.ts`），而是另開一條 Express 直連 `pg` 的路徑？程式碼內無註解說明，是為了繞過 RLS 看到全部資料，還是有其他效能/功能考量？
8. 全專案原始碼中完全沒有 `TODO`/`FIXME`/`HACK` 註解——是否有內部（例如 `.internal/docs/` 或 GitHub Issues）的技術債追蹤機制取代了程式碼內標記？是否有 lint 規則強制禁止合併含這類標記的 PR？
9. `docker-compose.yml` 中 Postgres container 啟動時帶的 `app.encryption_key` GUC，與 Node 端 `EncryptionManager`（`ENCRYPTION_KEY`/`JWT_SECRET` fallback 邏輯）是否為同一套加密機制，或分別服務不同的用途（例如 pgcrypto 的 column-level encryption vs 應用層 secrets 加密）？
10. AI Gateway 目前只有 OpenRouter 一個 concrete provider 且沒有共用 interface——這是刻意的架構決定（把多模型抽象完全外包給 OpenRouter），還是未來計畫直連其他 LLM vendor（例如 Anthropic/OpenAI 官方 API）時才會補上 `base.provider.ts`？

---

## 落差/發現數量統計小結

| 類別 | 數量 |
|---|---|
| 文件-程式碼落差 | 7（高 2／中 2／低 3） |
| TODO/FIXME/HACK/XXX 程式碼標記 | 0 |
| ⚠️ 未驗證項目（`_context/*.md` 彙整） | 30（見第 4 節分類） |
| 已知技術債項目 | 9 |
| 建議深入調查區域 | 9 |
| 建議向維護者確認的問題 | 10 |

**最值得優先處理的發現**：落差 #1（`CONTRIBUTING.md` 聲稱 backend 用 Better Auth，實際是完全自研的 JWT/PKCE 系統）。這不只是文字誤差——它會讓任何想貢獻 auth 相關功能的人一開始就找錯方向（去讀 Better Auth 官方文件而非本專案的 `TokenManager`/`OAuthPKCEService`），且暗示 `CONTRIBUTING.md` 這份對外文件可能已經一段時間沒有跟著程式碼演進更新，值得連帶檢查其餘段落（例如 Project Structure 落差 #2）是否也已過時。
