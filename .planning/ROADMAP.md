# Roadmap: Sirius Expression Framework (Phase 7b)

## Overview

Replace the Phase 7a `sirius::expression` PIMPL wrapper (around `duckdb::Expression`) with a Sirius-native AST under `sirius::ast::`, rewritten executor + translator, and a closed `function_id` enum. Delivered as 11 atomic, independently-mergeable PRs where every intermediate commit leaves `sirius_unittest`, `tpch-sirius.test`, and the `-DENABLE_LEGACY_SIRIUS=ON` build green with no TPC-H SF1 regression. Ends with a non-DuckDB frontend smoke test proving the decoupling. Upstream tracking: [sirius-db/sirius#666](https://github.com/sirius-db/sirius/issues/666).

## Phases

**Phase Numbering:**
- Integer phases (1–11): The 11 atomic deliverables.
- Each phase maps 1:1 to a GitHub sub-issue under [#666](https://github.com/sirius-db/sirius/issues/666) on sirius-db/sirius.

- [x] **Phase 1: AST Scaffold** ([#694](https://github.com/sirius-db/sirius/issues/694)) — Land `sirius::ast::node` variant + 9 node structs, zero callers. **Complete 2026-04-24** (rebased onto upstream `4e035090` after PR #693; full `sirius_unittest` + `[ast_scaffold]` suite green).
- [x] **Phase 2: Sirius Value Type** ([#695](https://github.com/sirius-db/sirius/issues/695)) — `sirius::value` variant + bidirectional `duckdb::Value` mappers. **Complete 2026-04-27** (4 atomic commits, full `sirius_unittest` + `[ast_value]` (27) + `[ast_scaffold]` (11) suites green).
- [x] **Phase 3: Function ID Enum** ([#696](https://github.com/sirius-db/sirius/issues/696)) — Closed `sirius::function_id` + name mappers; delete `*_FUNC_STR` macros and `supported_ast_functions` allowlist. **Complete 2026-04-27** (merged upstream as `f24fcadf`, PR [#716](https://github.com/sirius-db/sirius/pull/716)).
- [x] **Phase 4: DuckDB→Sirius Translator** ([#697](https://github.com/sirius-db/sirius/issues/697)) — `sirius::ast::from_duckdb(duckdb::Expression const&)` additive. (completed 2026-05-21)
- [x] **Phase 5: Dual-Path Executor** ([#698](https://github.com/sirius-db/sirius/issues/698)) — `gpu_expression_executor` gains Sirius-AST overloads alongside DuckDB-typed ones. (completed 2026-05-26 — 5 atomic commits 7ca1c593..2bbc37d3; full `sirius_unittest` 1,512/1,512 green)
- [ ] **Phase 6: Per-Specialization Migration** ([#699](https://github.com/sirius-db/sirius/issues/699)) — Flip each of the 9 `gpu_execute_*.cpp` files to take `sirius::ast::<node>` (9 atomic commits).
- [ ] **Phase 7: Translator Flip** ([#700](https://github.com/sirius-db/sirius/issues/700)) — `gpu_expression_translator` input type flips from `duckdb::Expression` to `sirius::ast::node`.
- [ ] **Phase 8: Wrapper Flip** ([#701](https://github.com/sirius-db/sirius/issues/701)) — `sirius::expression` PIMPL rewires to hold `std::unique_ptr<sirius::ast::node>`; plan builders call `ast::from_duckdb` at the boundary.
- [ ] **Phase 9: Remove DuckDB-Typed Executor Overloads** ([#702](https://github.com/sirius-db/sirius/issues/702)) — Delete the legacy executor methods; migrate test bodies to build Sirius AST directly.
- [ ] **Phase 10: Retire the Wrapper** ([#703](https://github.com/sirius-db/sirius/issues/703)) — Delete `sirius::expression` PIMPL; operators hold AST nodes directly.
- [ ] **Phase 11: Non-DuckDB Frontend Proof** ([#704](https://github.com/sirius-db/sirius/issues/704)) — End-to-end test that builds `sirius::ast::node` by hand and runs it through the executor.

## Phase Details

### Phase 1: AST Scaffold
**Goal**: Introduce the Sirius-native AST type hierarchy with zero callers — pure scaffolding PR.
**Depends on**: Nothing (first phase).
**Requirements**: AST-01
**Success Criteria** (what must be TRUE):
  1. `src/include/expression/ast/node.hpp` defines `sirius::ast::node = std::variant<reference, constant, comparison, conjunction, between, case_, cast, operator_, function_call>` with no `duckdb/*` include.
  2. Each of the 9 node structs has its own header under `src/include/expression/ast/` and forward-declares `node` for recursive children.
  3. Children are stored as `std::unique_ptr<node>` (or `std::vector<std::unique_ptr<node>>` for variadic nodes).
  4. Full `sirius_unittest` and `tpch-sirius.test` pass on both `-DENABLE_LEGACY_SIRIUS=OFF` and `ON`. No semantic change — no behavior shift.
**Plans**: 2 plans.

Plans:
- [ ] 01-01-PLAN.md — Land the 10 AST headers (node.hpp variant alias + 9 leaf node-struct headers) under src/include/expression/ast/.
- [ ] 01-02-PLAN.md — Add Catch2 unit test test/cpp/expression/test_ast_scaffold.cpp and register it in CMakeLists.txt TEST_SOURCES.

### Phase 2: Sirius Value Type
**Goal**: Introduce `sirius::value` as the Sirius-native equivalent of `duckdb::Value` for the AST constant payload.
**Depends on**: Phase 1.
**Requirements**: AST-02
**Success Criteria**:
  1. `src/include/expression/value.hpp` defines `sirius::value = std::variant<...>` keyed on the subset of `sirius::type_id` values the executor's constant path handles today.
  2. Bidirectional `value from_duckdb(duckdb::Value const&)` and `duckdb::Value to_duckdb(value const&)` live in a single `.cpp` that is the only DuckDB-including TU for this module.
  3. `sirius::ast::constant` holds a `sirius::value` payload (no callers yet).
  4. Unit tests cover all supported `sirius::type_id` variants round-tripping via `from_duckdb` / `to_duckdb`.
  5. Full suites green on both builds. No semantic change.
**Plans**: 1 plan.

Plans:
- [x] 02-01-PLAN.md — Land `sirius::value` + mappers + round-trip tests; wire the type into `sirius::ast::constant` (4 atomic commits + 1 user-verify checkpoint).

### Phase 3: Function ID Enum
**Goal**: Replace the 27 `*_FUNC_STR` string macros and the `supported_ast_functions` allowlist with a closed `sirius::function_id` enum.
**Depends on**: Phase 1.
**Requirements**: AST-03
**Success Criteria**:
  1. `src/include/expression/function_id.hpp` defines a closed `enum class sirius::function_id : uint16_t` with the 27 entries enumerated in PROJECT.md.
  2. `std::optional<function_id> from_duckdb_function_name(std::string_view)` and `std::string_view to_duckdb_function_name(function_id)` defined in a single `.cpp`.
  3. `supported_ast_functions` allowlist becomes `std::array<function_id, 6>` of `{add, sub, mul, div, int_div, mod}`. The `*_FUNC_STR` macros in `gpu_execute_function.cpp` are deleted.
  4. `gpu_execute_function.cpp` switches on `function_id` (from the still-DuckDB-typed input) instead of string compare.
  5. Full suites green on both builds. No semantic change — string match → enum switch is byte-equivalent.
**Plans**: 1 plan.

Plans:
- [x] 03-01-PLAN.md — Land `sirius::function_id` closed enum + name mappers + allowlist swap + `gpu_execute_function.cpp` dispatch refactor (4 atomic commits + 1 user-verify checkpoint). **Merged upstream as `f24fcadf` (PR #716) on 2026-05-19.**

### Phase 4: DuckDB→Sirius Translator
**Goal**: Ship `sirius::ast::from_duckdb(duckdb::Expression const&)` as the sole DuckDB→Sirius conversion point. No callers yet.
**Depends on**: Phases 1, 2, 3.
**Requirements**: TRANS-01, TEST-02
**Success Criteria**:
  1. `src/include/expression/ast/from_duckdb.hpp` declares the translator; `src/expression/ast/from_duckdb.cpp` implements it.
  2. All 9 `duckdb::Bound*Expression` classes are handled; unsupported shapes return `std::nullopt` (not throw), enabling fallback signalling.
  3. `test/cpp/expression/test_ast_from_duckdb.cpp` covers all 9 node kinds plus representative function and operator cases plus fallback negative cases.
  4. Full suites green on both builds. Translator is dead code at runtime — only exercised by the new test.
**Plans**: 1 plan.

Plans:
- [x] 04-01-PLAN.md — Land `sirius::ast::from_duckdb` translator (header + impl + Catch2 matrix + Binder E2E) and retire Phase 1's `detail::function_placeholder` in favor of `sirius::function_id` (5 atomic commits — 4 planned + 1 test-fix). **Complete 2026-05-21** (5 commits a74864d2 → ca15d25b; full sirius_unittest 1323/1323; [ast_from_duckdb] 33/33).

### Phase 5: Dual-Path Executor
**Goal**: `gpu_expression_executor` gains Sirius-AST overloads alongside the existing DuckDB-typed ones. Both paths coexist; callers still use the DuckDB path.
**Depends on**: Phase 4.
**Requirements**: EXEC-01 (partial — signature only)
**Success Criteria**:
  1. `gpu_expression_executor` declares `execute(sirius::ast::node const&, execution_mode)` and `count_ast_ops(sirius::ast::node const&)` alongside the existing DuckDB-typed versions.
  2. New overloads dispatch via `std::visit` and delegate to new `sirius::ast::<node>`-typed specialization helpers that internally call the existing DuckDB-typed specializations (shim layer).
  3. No call site migrated yet. The new path is hit only by new unit tests.
  4. Full suites green on both builds. Shim tests verify Sirius-AST inputs produce the same outputs as DuckDB-typed inputs on identical expressions.
**Plans**: 1 plan.

Plans:
- [x] 05-01-dual-path-executor-PLAN.md — Add to_duckdb translator + dual-path executor surface (header signatures + std::visit dispatcher + 11 per-alternative round-trip shims + _ast_expressions state) + three-leg byte-equivalence test (5 atomic commits + per-commit user-verify checkpoint). **Complete 2026-05-26** (5 commits 7ca1c593..2bbc37d3; full sirius_unittest 1,512/1,512; `[ast_to_duckdb]` 41/41; `[gpu_expression_executor_ast]` 20/20).

### Phase 6: Per-Specialization Migration
**Goal**: Flip each of the 9 `gpu_execute_*.cpp` files to take `sirius::ast::<node>` as the primary parameter. The DuckDB-typed specialization methods stay as shims until Phase 9.
**Depends on**: Phase 5.
**Requirements**: EXEC-01
**Success Criteria**:
  1. Each of `gpu_execute_between`, `_case`, `_cast`, `_comparison`, `_conjunction`, `_constant`, `_function`, `_operator`, `_reference` has its primary implementation take `sirius::ast::<node>`.
  2. The old DuckDB-typed specialization becomes a one-liner that calls `ast::from_duckdb(expr)` then delegates to the new implementation.
  3. Each of the 9 commits is individually revertable; the full suite passes at every commit.
  4. Per-specialization tests in `test_gpu_expression_executor.cpp` now exercise the Sirius-AST path directly; DuckDB-path tests remain green via the shim.
  5. Full suites green on both builds at each commit.
**Plans**: 1 plan with 9 atomic tasks (one per specialization), delivered as 9 commits or 9 PRs at author's discretion.

Plans:
- [ ] 06-01: Migrate all 9 specializations in commit order `reference → constant → comparison → conjunction → between → operator → cast → function → case`. Each commit green on the full suite.

### Phase 7: Translator Flip
**Goal**: `gpu_expression_translator` input type flips from `duckdb::Expression const&` to `sirius::ast::node const&`. Plan builders call `ast::from_duckdb` at the boundary before invoking the translator.
**Depends on**: Phase 6.
**Requirements**: EXEC-02
**Success Criteria**:
  1. `gpu_expression_translator::translate_expression` and `add_expression` overloads take `sirius::ast::node const&` parameters; internal dispatch uses `std::visit`.
  2. The 9 DuckDB-typed `add_expression(duckdb::Bound*Expression const&, ...)` overloads are deleted.
  3. `supported_ast_cast_types` converts from `std::array<duckdb::LogicalTypeId, 3>` to `std::array<sirius::type_id, 3>` (or equivalent).
  4. Plan builders in `src/planner/` that call the translator first construct a Sirius AST via `ast::from_duckdb`.
  5. Full suites green on both builds. No TPC-H SF1 regression across `materialize` / `ast_interpret` / `ast_jit`.
**Plans**: 1 plan.

Plans:
- [ ] 07-01: Flip translator input type, migrate plan-builder call sites, delete DuckDB-typed translator internals.

### Phase 8: Wrapper Flip
**Goal**: `sirius::expression` PIMPL rewires from `std::unique_ptr<duckdb::Expression>` to `std::unique_ptr<sirius::ast::node>`. The 35 boundary files unchanged externally.
**Depends on**: Phase 7.
**Requirements**: WRAP-01 (first half)
**Success Criteria**:
  1. `expression::impl` in `expression_internal.hpp` holds `std::unique_ptr<sirius::ast::node>`.
  2. `wrap(std::unique_ptr<duckdb::Expression>)` internally calls `ast::from_duckdb` and stores the resulting node.
  3. `unwrap(expression const&)` returns `sirius::ast::node const*` (breaking change to unwrap-callers).
  4. All 35 boundary files compile and run green after the flip. Any caller that was doing `sirius::unwrap(e)->Cast<duckdb::Bound*Expression>()` has been migrated to Sirius AST usage.
  5. Full suites green on both builds. No TPC-H SF1 regression.
  6. `sirius::join_condition::left` / `.right` become Sirius AST nodes.
**Plans**: 1 plan.

Plans:
- [ ] 08-01: Flip wrapper backing type + update all `unwrap()` consumers; migrate `sirius::join_condition`.

### Phase 9: Remove DuckDB-Typed Executor Overloads
**Goal**: Delete the 9 legacy DuckDB-typed specialization methods, the `count_ast_ops(duckdb::Expression)` overload, and the `gpu_expression_executor(duckdb::Expression const*)` ctor. Migrate the last test bodies to build Sirius AST directly.
**Depends on**: Phase 8.
**Requirements**: EXEC-03, TEST-01
**Success Criteria**:
  1. `gpu_expression_executor.hpp` has no `duckdb::BoundXxxExpression` forward declarations or DuckDB-typed overloads.
  2. `gpu_expression_executor.cpp` contains no `duckdb/planner/expression/*` includes.
  3. `test_gpu_expression_executor.cpp` (~2,022 LOC) builds all test expressions as `sirius::ast::node` directly; no `duckdb/planner/expression/*` in the test body. Coverage parity preserved (70/70 suite count unchanged).
  4. `test_gpu_expression_translator.cpp` (~1,823 LOC) migrated the same way.
  5. Full suites green on both builds. No TPC-H SF1 regression.
**Plans**: 1 plan with test migration split out as a sub-task.

Plans:
- [ ] 09-01: Delete DuckDB-typed executor surfaces; migrate `test_gpu_expression_executor.cpp` + `test_gpu_expression_translator.cpp` to Sirius AST.

### Phase 10: Retire the Wrapper
**Goal**: Delete the `sirius::expression` PIMPL. Operators hold the Sirius AST directly.
**Depends on**: Phase 9.
**Requirements**: WRAP-01 (second half)
**Success Criteria**:
  1. `src/include/expression/expression.hpp`, `src/include/expression/expression_internal.hpp`, `src/expression/expression.cpp` are deleted (or reduced to re-exports if needed during a transition commit).
  2. `wrap()`, `wrap_many()`, `unwrap()`, `release()` functions are removed.
  3. The 35 boundary files now hold `std::unique_ptr<sirius::ast::node>` (or `duckdb::vector<std::unique_ptr<sirius::ast::node>>`) directly.
  4. `sirius::join_condition` keeps its mirror shape but now relies on AST nodes directly.
  5. Full suites green on both builds. No TPC-H SF1 regression.
**Plans**: 1 plan.

Plans:
- [ ] 10-01: Delete wrapper + migrate all 35 boundary files to direct AST ownership.

### Phase 11: Non-DuckDB Frontend Proof
**Goal**: End-to-end integration test that builds a Sirius AST by hand (no `duckdb/*` includes anywhere in the test body) and runs it through `gpu_expression_executor` to produce a final cuDF column.
**Depends on**: Phase 10.
**Requirements**: PROOF-01
**Success Criteria**:
  1. A new test file under `test/cpp/integration/` or `test/cpp/expression/` constructs a non-trivial `sirius::ast::node` tree (at minimum: projection with reference + constant + binary function + cast).
  2. The test includes **no** `duckdb/*` headers anywhere in its body (verified by a grep guard in the test file).
  3. The test runs the expression through `gpu_expression_executor` and asserts final column values against a hand-constructed expected cuDF column.
  4. Full suites green on both builds. This test is the authoritative acceptance criterion for #666.
**Plans**: 1 plan.

Plans:
- [ ] 11-01: Author the non-DuckDB frontend smoke test + grep guard.

## Cross-Cutting Guarantees

These apply to every phase, not a phase of their own. Traced via REQ-ATOMIC-01:

1. Each phase's PR(s) keep `sirius_unittest` green (full suite) and `tpch-sirius.test` green on both `-DENABLE_LEGACY_SIRIUS=OFF` and `ON`.
2. No TPC-H SF1 regression across `materialize` / `ast_interpret` / `ast_jit` (measured at author's build-and-report step).
3. Pre-commit hooks pass with no `--no-verify`.
4. Each PR links to its upstream GitHub sub-issue.
5. Each PR is independently revertable.

## Progress

**Execution Order:** Phases execute in numeric order 1 → 11.

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. AST Scaffold | 0/1 | Not started | - |
| 2. Sirius Value Type | 0/1 | Not started | - |
| 3. Function ID Enum | 0/1 | Not started | - |
| 4. DuckDB→Sirius Translator | 1/1 | Complete    | 2026-05-21 |
| 5. Dual-Path Executor | 0/1 | Not started | - |
| 6. Per-Specialization Migration | 0/1 | Not started | - |
| 7. Translator Flip | 0/1 | Not started | - |
| 8. Wrapper Flip | 0/1 | Not started | - |
| 9. Remove DuckDB Overloads | 0/1 | Not started | - |
| 10. Retire the Wrapper | 0/1 | Not started | - |
| 11. Non-DuckDB Frontend Proof | 0/1 | Not started | - |

**Requirement Traceability:**

| Requirement | Phase | Status |
|-------------|-------|--------|
| AST-01 | Phase 1 | Pending |
| AST-02 | Phase 2 | Pending |
| AST-03 | Phase 3 | Complete |
| TRANS-01 | Phase 4 | Pending |
| TEST-02 | Phase 4 | Pending |
| EXEC-01 | Phase 5 + Phase 6 | Pending |
| EXEC-02 | Phase 7 | Pending |
| WRAP-01 | Phase 8 + Phase 10 | Pending |
| EXEC-03 | Phase 9 | Pending |
| TEST-01 | Phase 9 | Pending |
| PROOF-01 | Phase 11 | Pending |
| ATOMIC-01 | All phases (cross-cutting) | Pending |

Coverage: 12 / 12 requirements mapped to phases ✓

---
*Roadmap created: 2026-04-24*
