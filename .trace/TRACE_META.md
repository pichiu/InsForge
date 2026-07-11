# Trace Metadata

## 分支資訊
- **Base Branch**: main
- **Trace Branch**: claude/codebase-trace-documentation-jmwu2d

> ⚠️ 說明：本次執行環境要求所有 trace 產出直接在指定的工作分支 `claude/codebase-trace-documentation-jmwu2d` 上進行（而非另外建立 `trace/docs` 分支）。該分支於 base commit `0dd55c5` 由 `main` 分出，尚無其他非 trace 相關 commit，等效於獨立的 trace 分支。

## 最後 Trace 資訊
- **Base Commit Hash**: `0dd55c5fbcbe93be264cfd32d0cfd894cb2045f8`
- **日期**: 2026-07-11
- **Trace 類型**: full
- **涵蓋範圍**: 全部套件（`backend/`、`frontend/`、`packages/dashboard/`、`packages/ui/`、`packages/shared-schemas/`、`functions/`），使用者於 Stage 1.5 範圍評估後確認採用「完整 trace 全部套件」選項。

## 文件清單

| 文件 | 對應 Base Commit | 最後更新日期 |
|------|-----------------|-------------|
| INDEX.md | 0dd55c5 | 2026-07-11 |
| ARCHITECTURE.md | 0dd55c5 | 2026-07-11 |
| DATA_MODEL.md | 0dd55c5 | 2026-07-11 |
| API_SURFACE_part1.md | 0dd55c5 | 2026-07-11 |
| API_SURFACE_part2.md | 0dd55c5 | 2026-07-11 |
| DEV_GUIDE.md | 0dd55c5 | 2026-07-11 |
| CODEBASE_MAP.md | 0dd55c5 | 2026-07-11 |
| DISCOVERY_LOG.md | 0dd55c5 | 2026-07-11 |

## _context/ 保留狀態

保留（使用者於 Stage 4 確認保留，供未來增量更新 prompt 讀取以判斷影響範圍）：

- `_context/recon.md`
- `_context/web_findings.md`
- `_context/entry_points.md`
- `_context/data_flow.md`
- `_context/core_logic.md`
- `_context/extensions.md`
- `_context/integrations.md`
- `_context/configuration.md`

## 變更歷程

| 日期 | 類型 | Base Commit 範圍 | 更新的文件 | 摘要 |
|------|------|-----------------|-----------|------|
| 2026-07-11 | full | initial..0dd55c5 | 全部 | 初次 trace：涵蓋 InsForge monorepo 全套件（backend Express API、packages/dashboard、packages/ui、packages/shared-schemas、frontend self-hosting shell、functions Deno runtime）。共產出 8 份 `_context/*.md` 中間分析檔（1945 行）與 7 份最終文件（INDEX/CODEBASE_MAP/ARCHITECTURE/DATA_MODEL/API_SURFACE\*2/DEV_GUIDE/DISCOVERY_LOG）。發現 1 項高嚴重度文件落差：`CONTRIBUTING.md` 聲稱使用「Better Auth integration」，但程式碼實際為完全自研的 JWT/PKCE 認證系統，未使用 `better-auth` 套件。 |
