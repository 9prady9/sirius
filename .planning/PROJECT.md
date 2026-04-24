# Sirius Expression Framework (Phase 7b of DuckDB Decoupling)

## What This Is

A Sirius-native expression IR under `sirius::ast::` that replaces the Phase 7a PIMPL wrapper (`sirius::expression` around `duckdb::Expression`) with a proper AST type hierarchy mirroring the Phase 6 precedent of `sirius::logical_type` / `sirius::type_id`. The plan generator becomes the sole DuckDB→Sirius translator; the GPU expression executor and all specializations dispatch on Sirius AST nodes directly, with no `duckdb::Bound*Expression` references below the plan-builder boundary.

This closes the last semantic coupling to DuckDB in Super Sirius operator evaluation and is tracked upstream as GitHub issue [sirius-db/sirius#666](https://github.com/sirius-db/sirius/issues/666) (parent: [#556](https://github.com/sirius-db/sirius/issues/556), top-level: [#545](https://github.com/sirius-db/sirius/issues/545)).

## Core Value

**Every PR lands green on `tpch-sirius.test` and `sirius_unittest` (incl. the 70/70 expression-executor suite) with no TPC-H SF1 regression across `materialize` / `ast_interpret` / `ast_jit`.** Correctness and CI-green atomicity over speed of delivery.

## Requirements

### Validated

<!-- Inferred from the existing codebase -->

- ✓ Phase 0–6 (DuckDB compat header, Sirius-owned enums, Sirius exceptions, Sirius containers/idx_t, operator-state types, pipeline utilities, `sirius::logical_type` / `sirius::type_id` type system) — shipped across #549–#555.
- ✓ Phase 7a opaque `sirius::expression` PIMPL wrapper around `duckdb::Expression` and `sirius::join_condition` mirror struct — shipped as #665 (PRs #668 + #669).
- ✓ GPU expression executor path (`gpu_expression_executor`) with AST_INTERPRET / AST_JIT / MATERIALIZE strategies and 9 specializations — functional with DuckDB-typed input.
- ✓ `sirius::type_id` / `sirius::logical_type` ready to key the new constant payload representation.

### Active

- [ ] **REQ-AST-01** Define the Sirius-native AST type hierarchy (`sirius::ast::node` variant + 9 node structs: `reference`, `constant`, `comparison`, `conjunction`, `between`, `case_`, `cast`, `operator_`, `function_call`) with children stored via `std::unique_ptr`, no DuckDB includes in the public header.
- [ ] **REQ-AST-02** Define `sirius::value` — a type-safe variant keyed on `sirius::type_id` — to replace `duckdb::Value` as the payload of `sirius::ast::constant`.
- [ ] **REQ-AST-03** Define `sirius::function_id` as a closed enum covering the 27 scalar functions the GPU executor supports today (`add`, `sub`, `mul`, `div`, `int_div`, `mod`, `substring`, `like`, `not_like`, `contains`, `prefix`, `suffix`, `strlen`, `length`, `regexp_replace`, `year`, `month`, `day`, `hour`, `minute`, `second`, `millisecond`, `microsecond`, `date_trunc`, `row`, `struct_pack`, `error`), with bidirectional mapping helpers at the plan-builder boundary.
- [x] **REQ-TRANS-01** Implement the DuckDB-side translator `sirius::ast::from_duckdb(duckdb::Expression const&)` — the sole point that consumes `duckdb::Bound*Expression` types and produces Sirius AST nodes (including unsupported-shape → fallback signalling). **Validated in Phase 4** (2026-05-21, 6 commits a74864d2 → 1648c252; [ast_from_duckdb] 36/36; full sirius_unittest 1326/1326).
- [ ] **REQ-EXEC-01** Migrate `gpu_expression_executor::execute` + `count_ast_ops` + all 9 `gpu_execute_*.cpp` specializations to dispatch on `sirius::ast::node` via `std::visit` over the variant.
- [ ] **REQ-EXEC-02** Migrate `gpu_expression_translator` (DuckDB→cuDF-AST bridge) to consume `sirius::ast::node` instead of `duckdb::Expression`; plan builders call `ast::from_duckdb` first.
- [ ] **REQ-EXEC-03** Replace the `duckdb::LogicalTypeId`-based `supported_ast_cast_types` allowlist with a `sirius::type_id`-based one.
- [ ] **REQ-WRAP-01** Retire the Phase 7a PIMPL: either (a) redefine `sirius::expression` to hold a `std::unique_ptr<sirius::ast::node>`, or (b) remove the wrapper entirely and have operators hold the AST node directly. Decision deferred to the final migration phase.
- [ ] **REQ-TEST-01** Migrate the 3,845 lines of `test_gpu_expression_executor.cpp` + `test_gpu_expression_translator.cpp` to construct Sirius AST directly, while preserving coverage. Keep the tests green at every commit.
- [ ] **REQ-PROOF-01** Ship one end-to-end integration test that constructs a `sirius::ast::node` tree by hand (no `duckdb::*` includes) and runs it through the executor to final columns — proving frontend-independence.
- [ ] **REQ-ATOMIC-01** Every phase ships as one or more independently-mergeable PRs that keep `sirius_unittest` and `tpch-sirius.test` green on both default and `-DENABLE_LEGACY_SIRIUS=ON` builds, and do not regress TPC-H SF1 runtime across `materialize` / `ast_interpret` / `ast_jit`.

### Out of Scope

- **Closing existing fallbacks** (regex routing through the AST so the fallback checker stops matching on it; CASE in AST mode; COALESCE in AST mode) — this milestone is parity with 7a, not feature expansion. Tracked separately once the IR is in place.
- **Expression IR for the legacy `gpu_processing` path** — legacy lives in `src/legacy/expression_executor/` and does not use the 7a wrapper. The acceptance criterion from #665 applies here too: legacy builds (`-DENABLE_LEGACY_SIRIUS=ON`) stay green without changes.
- **Pluggable function registry** — rejected in favor of a closed `function_id` enum. A registry can be introduced later if non-DuckDB frontends require it; the enum is drop-in replaceable.
- **Replacing `duckdb::Value` in scan-operator parameter lists** (`sirius_physical_table_scan`, `_parquet_scan`, `_iceberg_scan`, `_duckdb_scan`) — those are table parameters, not expression payloads, and are out of scope for Phase 7.
- **Replacing `duckdb::Value` in plan-time statistics** (`sirius_plan_aggregate.cpp`'s `NumericStats` calls) — outside the expression subsystem.
- **Substrait or other external IR interop** — Sirius AST is the internal IR. External IRs, if any, translate to Sirius AST at the same boundary as DuckDB.
- **Refactoring cuDF AST consumption inside the translator** — the `cudf::ast::tree` output format and its scalar-lifetime handling stays as-is.
- **Changes to operator semantics, memory reservations, or pipeline wiring** — this is purely an IR refactor. Operators continue to call `gpu_expression_executor` the same way.

## Context

**Codebase state.** Sirius is a DuckDB extension (`sirius.duckdb_extension`) routing supported SQL ops to GPU execution via cuDF/RMM/cuCascade. The active "Super Sirius" path lives in `namespace sirius`; a legacy `gpu_processing` path is isolated under `src/legacy/` and does not share the Phase 7 wrapper. The Phase 7a migration (tracked as issue [#669](https://github.com/sirius-db/sirius/issues/669)) completed in March–April 2026 and landed the PIMPL wrapper at the operator/plan-generator/executor boundary, leaving DuckDB expression types intact below it. Extensive architectural docs in `docs/super-sirius/`; codebase map in `.planning/codebase/`.

**Why this matters.** #666 is the last block before issue [#557](https://github.com/sirius-db/sirius/issues/557) (Phase 8: directory restructuring into an adapter layer) and unblocks non-DuckDB frontends. Every other phase of the #545 tracking issue has closed; 7b is the final semantic decoupling.

**Scale.** ~60 files touched, ~7,500 LOC of semantic churn (including tests). Breakdown:
- 8 new headers in `src/include/expression/ast/`, plus `value.hpp` and `function_id.hpp`
- 11 executor files rewritten (header + impl + 9 specializations) = ~1,900 LOC
- 2 translator files rewritten (header + impl) = ~990 LOC
- 35 boundary files keep their `sirius::expression` usage but gain semantic meaning once the wrapper flips
- 2 test files migrated = ~3,845 LOC
- 3 wrapper files deleted = ~215 LOC

**Constraints from the issue acceptance criteria.**
- Build must pass on both `ENABLE_LEGACY_SIRIUS=OFF` and `ON` at every commit.
- Full `sirius_unittest` (including the 70/70 expression-executor suite) green at every commit.
- `tpch-sirius.test` green on release at every commit.
- No TPC-H SF1 regression across `materialize` / `ast_interpret` / `ast_jit`.

**Design choices locked (see Key Decisions below).**
- AST dispatch: `std::variant<...>` + `std::visit` (not class hierarchy + virtual, not enum + switch+downcast).
- Constant payload: `std::variant` keyed by `sirius::type_id`.
- Function identification: closed `sirius::function_id` enum.
- Scope: parity with 7a + one non-DuckDB frontend smoke test.
- Delivery: atomic, independently-mergeable PRs; the engine works at every commit.

## Constraints

- **Tech stack**: C++20, CUDA 13+ via pixi environment. No new runtime dependencies. `std::variant` and `std::visit` are standard library (C++17) — available. Builds via `make` (Makefile wrapping `extension-ci-tools`); CMake direct is not supported.
- **Compatibility**: Must build on both default and `-DENABLE_LEGACY_SIRIUS=ON`. Legacy path (`src/legacy/`) left untouched.
- **Correctness**: `sirius_unittest` and `tpch-sirius.test` green at every commit. TPC-H SF1 no-regression across all three executor strategies.
- **Performance**: No TPC-H SF1 regression across `materialize` / `ast_interpret` / `ast_jit`. An IR traversal that used to walk `duckdb::Expression` now walks `sirius::ast::node`; the variant must stay cache-friendly.
- **Upstream tracking**: Work lands upstream on `sirius-db/sirius`. PRs link to the Phase 7b sub-issues (to be created on sirius-db/sirius as sub-issues of #666).
- **No hooks, no `--no-verify`**: Pre-commit hooks (clang-format, clang-tidy with `WarningsAsErrors: '*'`, codespell, cmake-format) must pass.
- **Build is user-driven**: Per session convention, build + `sirius_unittest` execution happens on the developer's machine and is reported back, not run by the agent.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| AST dispatch via `std::variant` + `std::visit` | Type-safe, no virtuals, compile-time exhaustive. Caveats understood (recursive children via `unique_ptr`, variant size = largest alt, compile-time cost trivial at N=9, debugger ergonomics acceptable). | — Pending |
| Constant payload via `std::variant` keyed on `sirius::type_id` | Type-safe, mirrors the type system's precedent, easy to translate to/from `duckdb::Value` at the boundary. | — Pending |
| Function IDs as a closed `sirius::function_id` enum | Executor supports exactly 27 named functions today; enum makes dispatch a switch, avoids registry plumbing. Extensible later if non-DuckDB frontends need user-defined functions. | — Pending |
| v1 scope: 7a parity + one non-DuckDB smoke test | Ship decoupling without bundling feature work (regex, CASE, COALESCE in AST). Keeps PR reviews tractable. | — Pending |
| Delivery: atomic, independently-mergeable phases | Prior 7a migration (issue #669) proved single-PR mega-refactors invite rework; each sub-issue below is one or a few PRs that leaves the engine working. | — Pending |
| Wrapper retirement: flip PIMPL to hold Sirius AST first, delete wrapper only after all callers rely on it natively | Keeps the 35 boundary files unchanged during the executor migration; the final wrapper-delete PR becomes a focused cleanup. | — Pending |
| Test migration strategy: rewrite tests to build Sirius AST directly (not a DuckDB→Sirius test helper) | A test helper still depends on DuckDB includes, contradicting the frontend-independence promise. Direct Sirius-AST tests also exercise the non-DuckDB frontend path as a byproduct. | — Pending |
| PR-per-phase strategy: stack all 11 phases on `sirius_expression_framework` (this worktree), extract clean per-phase PR branches via `/gsd-pr-branch <target> <name>` | Don't block on upstream review between phases. Each PR branch carries only the commits relevant to its phase (proven by #707 = `sirius_ast_scaffold` extracted from this branch). Dev branch accumulates planning + code history; PR branches stay code-only and reviewable. | Locked 2026-04-27 |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-05-21 after Phase 4 (REQ-TRANS-01) validated*
