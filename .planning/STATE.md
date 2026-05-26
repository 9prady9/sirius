---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: executing
stopped_at: Phase 5 plan 1 complete — 5 atomic commits on sirius_expression_framework
last_updated: "2026-05-26T13:37:00Z"
last_activity: 2026-05-26 -- Phase 5 plan 1 complete (5 commits a74864d2..2bbc37d3 superseded; new HEAD 2bbc37d3)
progress:
  total_phases: 11
  completed_phases: 5
  total_plans: 7
  completed_plans: 5
  percent: 45
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-04-24)

**Core value:** Every PR lands green on `sirius_unittest` (incl. 76/76 expression-executor suite + 20/20 new gpu_expression_executor_ast suite). Legacy `gpu_processing` path (and `tpch-sirius.test`, which exercises it) is frozen — no Phase 7b commit touches `src/legacy/`.
**Current focus:** Phase 6 — Per-Specialization Migration (#699) — next to plan

## Current Position

Phase: 5 (Dual-Path Executor) — COMPLETE
Plan: 1 of 1 — DONE
Next phase: 6 — Per-Specialization Migration (#699) — ready to plan via `/gsd-discuss-phase 6`
Status: Phase 5 complete; awaiting Phase 6 discussion
Last activity: 2026-05-26 -- Phase 5 plan 1 complete; 5 atomic commits on sirius_expression_framework (HEAD: 2bbc37d3)

Progress: [█████░░░░░] 45% (5 of 11 phases)

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

- Total plans completed: 7 (init scaffolding + 01-01 + 02-01 + 03-01 + 04-01 + 05-01)
- Average duration: —
- Total execution time: — (per-plan timing tracked from Phase 5; Phase 5 plan 1 = ~119 min)

**By Phase:**

| Phase | Plans | Status | Notes |
|-------|-------|--------|-------|
| 1. AST Scaffold | 1/1 | ✅ COMPLETE | PR #707 (sirius_ast_scaffold) — open as draft upstream |
| 2. sirius::value  | 1/1 | ✅ COMPLETE | PR #715 (sirius_value) — open as draft upstream |
| 3. function_id enum | 1/1 | ✅ COMPLETE | PR #716 (sirius_function_id) — merged upstream as f24fcadf |
| 4. ast::from_duckdb | 1/1 | ✅ COMPLETE | Merged upstream as 8231f095 (PR #796) on 2026-05-21 |
| 5. dual-path executor | 1/1 | ✅ COMPLETE | 5 atomic commits 7ca1c593..2bbc37d3 on sirius_expression_framework |
| 6. per-specialization migration | 0/? | ⏳ NEXT | awaiting `/gsd-discuss-phase 6` |
| 7–11 | 0/? each | not yet planned | |

**Per-plan metrics:**

| Phase | Plan | Duration | Tasks | Files | Notes |
|-------|------|----------|-------|-------|-------|
| 5 | 1 | ~119 min | 5 | 7 | 4 functional commits + 1 style commit; 41 + 20 new TEST_CASEs; full sirius_unittest 1512/1512 |

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
- **Phase 5 (2026-05-26):** Per-alternative `to_duckdb(<alt>)` public overloads — declared in `to_duckdb.hpp`, called directly by executor shims to sidestep node's move-only constraint. Avoids a deep-clone helper.
- **Phase 5 (2026-05-26):** `function_call`'s `ScalarFunction` stub is name + empty arg-types + return_type + nullptr function pointer. `gpu_execute_function.cpp` dispatches by name → function_id, never reads ScalarFunction internals.
- **Phase 5 (2026-05-26):** `select(table_view)` AST branch round-trips through `to_duckdb` to recover return_type for the BOOLEAN debug-only D_ASSERT — matches DuckDB-branch parity with minimum semantic delta.
- **Phase 5 (2026-05-26):** `duckdb::unique_ptr<T>` ≠ `std::unique_ptr<T>` — they share `std::default_delete` but are not implicitly convertible. `to_duckdb.cpp` carries two anonymous-namespace adapters (`to_duck_ptr`, `from_duck_derived_ptr`) that release+wrap at the DuckDB ctor boundary. Pattern reusable for any future Sirius surface that returns `std::unique_ptr<duckdb::Expression>` but internally calls DuckDB ctors.

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

Last session: 2026-05-26T13:37:00Z
Stopped at: Phase 5 plan 1 complete — 5 atomic commits on sirius_expression_framework
Resume file: None

**Last commits (Phase 5, on `sirius_expression_framework`):**

- 2bbc37d3 — style(05-01): apply pre-commit hook fixups
- 930dccdc — test(05-01): three-leg ast equivalence test for dual-path executor
- 332aec16 — feat(05-01): implement per-alternative round-trip shims via to_duckdb
- 556b3b6f — feat(05-01): scaffold dual-path gpu_expression_executor surface
- 7ca1c593 — feat(05-01): add sirius::ast::to_duckdb translator and direct-ctor tests

**Verification (2026-05-26):**

- Build: clean, no warnings
- `[ast_to_duckdb]` (new): 134 assertions / 41 cases — PASS
- `[gpu_expression_executor_ast]` (new): 84 assertions / 20 cases — PASS
- `[expression_executor]`: 3,255 assertions / 76 cases — PASS (grew from 70 unrelated to Phase 5)
- `[expression_translator]`: 169 assertions / 54 cases — PASS
- `[ast_from_duckdb]` (Phase 4 regression gate): 139 assertions / 36 cases — PASS
- `[ast_function_id]` (Phase 3 regression gate): 88 assertions / 31 cases — PASS
- `[ast_value]` (Phase 2 regression gate): 84 assertions / 27 cases — PASS
- `[ast_scaffold]` (Phase 1 regression gate): 50 assertions / 16 cases — PASS
- Full `sirius_unittest`: 41,431,375 assertions / 1,512 cases — PASS
- Pre-commit: all hooks clean post style commit (clang-format, codespell, cmake-format).
- `src/legacy/` and `test/sql/tpch-sirius.test`: byte-unchanged (legacy-frozen invariant honored).
- `tpch-sirius.test` (SQLLogicTest): SKIPPED — exercises legacy `gpu_processing` path which Phase 7b never touches (per `feedback_legacy_sirius_frozen.md`).

**Next Phase:** 6 (Per-Specialization Migration — #699) — awaiting `/gsd-discuss-phase 6`

**Recommended Phase 6 starting point:** `gpu_execute_reference.cpp` — the leaf with no recursive children; the shim becomes "rewrite to call existing `execute(BoundReferenceExpression, mode)` with the Sirius `reference.column_index` field directly" rather than translate-then-traverse.
