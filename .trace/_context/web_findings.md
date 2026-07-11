# 線上資源搜尋結果 (Web Findings)

## 搜尋 1：「InsForge open source backend platform agentic coding MCP server」

- [GitHub - InsForge/InsForge](https://github.com/InsForge/insforge) — 官方 repo，slogan：「The all-in-one, open-source backend platform for agentic coding」。
- [Overview - InsForge Docs](https://docs.insforge.dev/introduction) — 官方產品文件首頁。
- [InsForge - The agent-native cloud infrastructure platform](https://insforge.dev/) — 官網首頁。
- [InsForge: An open-source Heroku-like platform for coding AI agents - GIGAZINE](https://gigazine.net/gsc_news/en/20260614-insforge/) — 第三方媒體報導，將 InsForge 類比為「給 AI coding agent 用的 Heroku」。
- [InsForge - The Backend Framework Built for Agentic Applications - Groundy](https://groundy.com/articles/insforge-backend-framework-built-specifically-agentic/)

**關鍵摘要**：InsForge 是給 AI coding agent（而非人類）用的後端開發平台，作為 agent 與後端 primitives（資料庫、認證、儲存、edge functions、model gateway）之間的「語意層」。MCP Server 支援 Cursor、Claude Code、GitHub Copilot、Google Antigravity、Codex、Cline、Windsurf、Kiro、Trae、Qoder、Roo Code 等主流 agent 開發環境。Apache 2.0 授權，原始碼公開。

## 搜尋 2：「InsForge.dev architecture blog announcement backend as a service」

- [InsForge 2.0 Launch](https://insforge.dev/blog/insforge-launch-v2) — 官方部落格，2.0 版發布公告。
- [InsForge Launch](https://insforge.dev/blog/insforge-launch) — 初版發布公告。
- [DeepWiki: InsForge/InsForge](https://deepwiki.com/InsForge/InsForge) — 第三方自動生成的程式碼 wiki，可作為交叉驗證來源（⚠️ 未逐頁核對，內容由第三方 LLM 生成，非官方）。
- [InsForge: The Postgres Backend for Coding Agents - AIToolly](https://aitoolly.com/ai-news/article/2026-05-08-insforge-a-comprehensive-postgres-based-backend-and-ai-gateway-for-coding-agents)
- [Superpowers vs Insforge 比較文章 (2026)](https://pasqualepillitteri.it/en/news/1341/superpowers-vs-insforge-comparison-2026) — 定位比較：InsForge 屬於「AI 後端」類別，而非「agentic framework」類別。

**關鍵摘要**：
- InsForge 由 Y Combinator 孵化（P26 batch，2026 春季梯次）。⚠️ 未驗證（來自第三方報導，非官方一手資料）。
- 對比 Supabase：InsForge 透過 MCP Server 直接暴露結構化 schema metadata（含 RLS policy、record count），讓 agent 不需多輪探索式往返即可操作後端；宣稱比 Supabase 快 1.6 倍、省 2.4 倍 token。⚠️ 未驗證（第三方測評數據，未見官方 benchmark 方法論）。
- 核心 building blocks：PostgreSQL + pgvector（向量搜尋）、S3 相容儲存、OAuth 認證、Deno-based edge functions、OpenAI 相容 model gateway（多 LLM provider）、realtime sync — 與本次程式碼掃描結果一致。
- 2.0 版本（相對於 2025-11 首次發布）：資料庫建立數成長 500%，近 99% 的操作/請求來自 agent 而非人類；provisioning、config、migration、runtime 操作皆由 agent 執行，不透過 dashboard/CLI/手寫腳本。

## 未執行的搜尋方向（因時間/規模考量略過，供後續補充）

- GitHub Discussions / Issues 中 pinned 或 architecture 相關討論串（未搜尋，建議後續針對特定子系統如 Compute/Realtime 補搜）。
- 官方 Discord / 社群頻道內容（無法直接搜尋私有頻道內容）。
- 與 Supabase / Firebase 等競品的完整技術比較文章（僅有第二手摘要，未深入）。

> ⚠️ 本次為大型 monorepo 全套件 trace，Web search 依 Stage 1 指引之「小型專案不必硬搜」原則，執行 2 輪關鍵搜尋取得專案定位與市場脈絡即止；核心技術細節以程式碼與 repo 內既有文件為準。
