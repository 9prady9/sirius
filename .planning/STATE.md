---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: ready_to_plan
stopped_at: Phase 4 context gathered (translator design + 6 locked decisions)
last_updated: "2026-05-21T10:53:08.127Z"
last_activity: 2026-05-21 -- Phase 4 execution started
progress:
  total_phases: 11
  completed_phases: 3
  total_plans: 5
  completed_plans: 3
  percent: 27
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-04-24)

**Core value:** Every PR lands green on `sirius_unittest` (incl. 70/70 expression-executor suite). Legacy `gpu_processing` path (and `tpch-sirius.test`, which exercises it) is frozen — no Phase 7b commit touches `src/legacy/`.
**Current focus:** Phase 4 — DuckDB→Sirius Translator

## Current Position

Phase: 5
Plan: Not started
Next phase: 4 — `sirius::ast::from_duckdb` translator (additive, #697) — ready to plan via `/gsd-discuss-phase 4`
Status: Ready to plan
Last activity: 2026-05-21

Progress: [███░░░░░░░] 27% (3 of 11 phases)

## Upstream Tracking

Tracked in GitHub issue [sirius-db/sirius#666](https://github.com/sirius-db/sirius/issues/666) (parent: [#556](https://github.com/sirius-db/sirius/issues/556), top-level: [#545](https://github.com/sirius-db/sirius/issues/545)).

Each of the 11 phases maps to one GitHub sub-issue under #666:

| Phase | Issue | Title |
|-------|-------|-------|
| 1  | [#694](https://github.com/sirius-db/sirius/issues/694) | AST type hierarchy scaffold |
| 2  | [#695](https://github.com/sirius-db/sirius/issues/695) | sirius::value for AST constant payload |
| 3  | [#696](https://github.com/sirius-db/sirius/issues/696) | closed sirius::function_id enum |
| 4  | [#697](https://github.com/sirius-db/sirius/issues/697) | sirius::ast::from_duckdb translator (additive) |
| 5  | [#698](https://github.com/sirius-db/sirius/issues/698) | dual-path gpu_expression_executor |
| 6  | [#699](https://github.com/sirius-db/sirius/issues/699) | migrate 9 gpu_execute_* specializations |
| 7  | [#700](https://github.com/sirius-db/sirius/issues/700) | flip gpu_expression_translator input |
| 8  | [#701](https://github.com/sirius-db/sirius/issues/701) | flip sirius::expression PIMPL to hold AST |
| 9  | [#702](https://github.com/sirius-db/sirius/issues/702) | remove DuckDB-typed executor overloads |
| 10 | [#703](https://github.com/sirius-db/sirius/issues/703) | retire the sirius::expression PIMPL wrapper |
| 11 | [#704](https://github.com/sirius-db/sirius/issues/704) | non-DuckDB frontend smoke test proof |

All 11 sub-issues are OPEN, labeled `duckdb`, assigned to `@9prady9`, and linked as sub-issues of #666 via the GitHub sub-issue API.

## Performance Metrics

**Velocity:**

- Total plans completed: 5 (init scaffolding + 01-01 + 02-01 + 03-01)
- Average duration: —
- Total execution time: — (per-plan timing not tracked yet)

**By Phase:**

| Phase | Plans | Status | Notes |
|-------|-------|--------|-------|
| 1. AST Scaffold | 1/1 | ✅ COMPLETE | PR #707 (sirius_ast_scaffold) — open as draft upstream |
| 2. sirius::value  | 1/1 | ✅ COMPLETE | PR #715 (sirius_value) — open as draft upstream |
| 3. function_id enum | 1/1 | ✅ COMPLETE | PR #716 (sirius_function_id) — open as draft upstream, base=dev, narratively stacked on #715 |
| 4. ast::from_duckdb | 0/? | ⏳ NEXT | awaiting `/gsd-discuss-phase 4` |
| 5–11 | 0/? each | not yet planned | |

*Updated after each plan completion*

## Accumulated Context

### Decisions

Logged in PROJECT.md Key Decisions table. Summary of choices made during initialization:

- AST dispatch via `std::variant` + `std::visit` (not class hierarchy + virtual, not enum + downcast).
- Constant payload via `std::variant` keyed on `sirius::type_id`.
- Function IDs as a closed `sirius::function_id` enum (27 entries).
- Scope: 7a parity + one non-DuckDB frontend smoke test (out: closing existing fallbacks).
- Delivery: 11 atomic, independently-mergeable PRs — engine green at every commit.
- Wrapper retirement split into two phases (Phase 8 flips backing type, Phase 10 deletes the wrapper).
- Tests migrated to build Sirius AST directly, not via a DuckDB helper.
- **PR strategy (locked 2026-04-27):** All 11 phases land sequentially on `sirius_expression_framework` (this worktree). Per-phase PR branches are extracted via `/gsd-pr-branch <target> <name>` from the relevant commits, as proven by #707 (`sirius_ast_scaffold`). No stacking, no waiting on upstream review between phases.

### Pending Todos

None yet.

### Blockers/Concerns

- Two earlier `.planning/` commits (`a1ae8906 docs: initialize project`, `90151e9f docs: map existing codebase`) landed before `commit_docs=false` was configured. Currently tracked in git history; user to decide whether to squash/untrack later.

## Deferred Items

| Category | Item | Status | Deferred At |
|----------|------|--------|-------------|
| Fallbacks | CASE in AST mode (v2 FB-01) | Deferred | 2026-04-24 |
| Fallbacks | COALESCE in AST mode (v2 FB-02) | Deferred | 2026-04-24 |
| Fallbacks | ~~Route `regexp_replace` through AST~~ | Retired 2026-04-24 | Legacy-only `FallbackChecker` (PR #693); regex is already GPU-executed via cuDF. |
| Extensibility | Pluggable function registry (v2 EXT-01) | Deferred | 2026-04-24 |
| Extensibility | Stable Sirius AST API for other frontends (v2 EXT-02) | Deferred | 2026-04-24 |

## Session Continuity

Last session: --stopped-at
Stopped at: Phase 4 context gathered (translator design + 6 locked decisions)
Resume file: --resume-file

**Last commits (Phase 3, on `sirius_expression_framework`):**

- 91ef997a — style(03-01): apply clang-format to phase 3 files
- f8634315 — refactor(03-01): switch gpu_execute_function to function_id enum dispatch
- 36a6a8de — test(03-01): add round-trip + alias + unknown-name tests for sirius::function_id
- 36ef9163 — feat(03-01): add sirius::function_id closed enum and name-mapper decls

**Verification (2026-04-27):**

- Build: clean, no warnings
- `[ast_function_id]`: 88 assertions / 31 cases — PASS
- `[ast_value]` (Phase 2 regression gate): 84 assertions / 27 cases — PASS
- `[ast_scaffold]` (Phase 1 regression gate): 44 assertions / 13 cases — PASS (grew from 11 due to `ebf52bf9` operator_ split, not a regression)
- Full `sirius_unittest`: 79,392,109 assertions / 1060 cases — PASS
- `tpch-sirius.test` (SQLLogicTest): SKIPPED — exercises legacy `gpu_processing` path which Phase 7b never touches (per `feedback_legacy_sirius_frozen.md`).

**Next Phase:** 4 (`sirius::ast::from_duckdb` translator, additive — #697) — awaiting `/gsd-discuss-phase 4`

**Planned Phase:** 4 (DuckDB→Sirius Translator) — 1 plans — 2026-05-21T10:42:06.634Z
