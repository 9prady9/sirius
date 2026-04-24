# Codebase Concerns

**Analysis Date:** 2026-05-20

This document captures risks, debt, and care-areas for the **Sirius Expression
Framework** milestone (Phase 7b — `sirius::ast::*` IR migration, upstream
tracking issue [sirius-db/sirius#666](https://github.com/sirius-db/sirius/issues/666)).
It is a **refresh** of the 2026-04-24 audit and supersedes it.

Phases 1–3 (AST scaffold, `sirius::value`, `function_id` enum) have merged
upstream; phases 4–11 (translator, dual-path executor, per-specialization
migration, translator flip, PIMPL flip, wrapper retirement, smoke test)
remain — see `.planning/STATE.md` and `.planning/ROADMAP.md`.

---

## Frozen Legacy Path

The legacy `gpu_processing` engine (entry point `CALL gpu_processing('...')`,
`namespace duckdb`) is **ring-fenced** for the duration of this milestone. No
Phase 7b PR is expected to touch any file below.

**Don't-touch list:**

- `src/legacy/` — all `.cpp`/`.cu` files (cpu_cache, gpu_buffer_manager,
  gpu_executor, gpu_meta_pipeline, gpu_physical_operator, gpu_pipeline,
  gpu_query_result, gpu_columns, gpu_context, gpu_physical_plan_generator,
  `operator/`, `plan/`, `cuda/`, `expression_executor/`).
- `src/legacy/fallback.cpp` — legacy `FallbackChecker` in `namespace duckdb`
  (recursive expression walker that throws on `regexp_replace`); only reached
  when `gpu_processing` is invoked.
- `src/include/legacy/` — all legacy headers (`gpu_physical_operator.hpp`,
  `gpu_executor.hpp`, `gpu_buffer_manager.hpp`, `cpu_cache.hpp`,
  `fallback.hpp`, `operator/`, etc.).
- `src/legacy/CMakeLists.txt` — the `ENABLE_LEGACY_SIRIUS` option (default
  `OFF`); legacy sources are only compiled when this is `ON`. PRs must keep
  `-DENABLE_LEGACY_SIRIUS=ON` builds green by NOT changing legacy files.

**Broken / skipped tests:**

- `test/sql/tpch-sirius.test` — SQLLogicTest. Every query uses
  `call gpu_processing("...")`, e.g. `test/sql/tpch-sirius.test:728` etc.,
  which exclusively exercises the legacy path. **Skipped during this
  milestone** per project convention — verified `PASS` recorded in STATE.md
  as "SKIPPED — exercises legacy `gpu_processing` path which Phase 7b never
  touches" (Plan 03-01 verify, 2026-04-27).
- `test/sql/clickbench-sirius.test` — same `gpu_processing` pattern;
  treat as skipped/unverified for this milestone. *Unverified — needs human
  review whether this is actively run in CI.*
- `test/cpp/memory_management/test_cpu_cache.cpp` — listed in
  `SIRIUS_LEGACY_TEST_SOURCES` (`src/legacy/CMakeLists.txt:106-108`); only
  built when `ENABLE_LEGACY_SIRIUS=ON`.

**Why ring-fenced:** PR #693 (4e035090 `Refactor legacy source cleanup`) and
PR #692 (03134b9c `refactor: isolate legacy CUDA under src/legacy/`) isolated
the legacy code; Phase 7b's acceptance contract is "no behavior change to
`-DENABLE_LEGACY_SIRIUS=ON` builds." Avoid touching legacy unless explicitly
in a "legacy-only" PR coordinated separately upstream.

**EOL signalling:** None in code. CLAUDE.md and `docs/super-sirius/README.md`
describe legacy as "the new task-based GPU execution engine ... replaces the
legacy `gpu_processing` path" but no commented EOL date or compile-time
`#warning`. Consider adding a sunset banner. *Unverified — needs human
review.*

---

## Active Refactor Risks (Phase 7b, sub-issues #697–#704)

The remaining 8 phases each touch the hot path of expression evaluation; any
intermediate commit must keep `sirius_unittest` (including the 70/70
expression-executor suite) green and TPC-H SF1 latency unregressed across
`materialize` / `ast_interpret` / `ast_jit` (REQ-ATOMIC-01).

### Phase 4 — `sirius::ast::from_duckdb` translator ([#697](https://github.com/sirius-db/sirius/issues/697))

**Where it lands:** new `src/include/expression/ast/from_duckdb.hpp` +
`src/expression/ast/from_duckdb.cpp`.

**Risk:** This is the sole boundary between DuckDB expression IR and Sirius
AST. Every `duckdb::Bound*Expression` subclass must be handled
(BoundReferenceExpression, BoundConstantExpression, BoundComparisonExpression,
BoundConjunctionExpression, BoundBetweenExpression, BoundCaseExpression,
BoundCastExpression, BoundOperatorExpression, BoundFunctionExpression) plus
unsupported shapes must return `std::nullopt` for fallback signalling, not
throw. A throw here cascades into the runtime fallback path
(`run_internal_cpu_fallback_query`) instead of giving a clean error.

**Care areas:**
- `duckdb::Value` → `sirius::value` round-trip exists in `src/expression/value.cpp`
  but is **only exercised by `[ast_value]` unit tests** — translator usage is
  the first real-world stress test.
- `duckdb` function-name lookup uses `from_duckdb_function_name` in
  `src/expression/function_id.cpp` (Phase 3); unknown names should signal
  fallback, not throw.

### Phase 5 — Dual-path `gpu_expression_executor` ([#698](https://github.com/sirius-db/sirius/issues/698))

**Risk:** Both DuckDB-typed and Sirius-AST overloads coexist as a shim layer.
Bugs here surface as silent divergence between the two paths under identical
inputs. Equivalence tests must cover every node kind, not just the easy ones.

**Files at risk:**
- `src/expression_executor/gpu_expression_executor.cpp` (475 LOC) —
  currently DuckDB-only; will gain `std::visit`-based dispatch.
- `src/include/expression_executor/gpu_expression_executor.hpp` — adds
  overloads alongside the existing `duckdb::Expression`-typed entry points.

### Phase 6 — Per-specialization migration ([#699](https://github.com/sirius-db/sirius/issues/699))

**Risk:** Flipping each of the 9 `gpu_execute_*.cpp` files one at a time.
The order is prescribed (`reference → constant → comparison → conjunction →
between → operator → cast → function → case`) and each commit must leave the
full suite green. A mid-flip revert is the recovery path; if more than one
commit becomes unrevertable the discipline of REQ-ATOMIC-01 collapses.

**Files at risk:**
- `src/expression_executor/specializations/gpu_execute_between.cpp`
- `src/expression_executor/specializations/gpu_execute_case.cpp`
- `src/expression_executor/specializations/gpu_execute_cast.cpp`
- `src/expression_executor/specializations/gpu_execute_comparison.cpp`
- `src/expression_executor/specializations/gpu_execute_conjunction.cpp`
- `src/expression_executor/specializations/gpu_execute_constant.cpp`
- `src/expression_executor/specializations/gpu_execute_function.cpp`
- `src/expression_executor/specializations/gpu_execute_operator.cpp`
- `src/expression_executor/specializations/gpu_execute_reference.cpp`

### Phase 7 — Translator flip ([#700](https://github.com/sirius-db/sirius/issues/700))

**Risk:** `gpu_expression_translator` (725 LOC) is the cuDF AST bridge;
changing its input type from `duckdb::Expression const&` to
`sirius::ast::node const&` is a flip-day for all plan-builder call sites in
`src/planner/sirius_plan_*.cpp` (17 files). Also flips
`supported_ast_cast_types` from a `std::array<duckdb::LogicalTypeId, 3>` to a
`sirius::type_id`-keyed array — currently still DuckDB-typed at
`src/include/expression_executor/ast_supported_types.hpp:35-36`.

### Phase 8 — Wrapper flip ([#701](https://github.com/sirius-db/sirius/issues/701))

**Risk:** `sirius::expression` PIMPL is the load-bearing wrapper at the
operator / plan-builder boundary. Flipping it from
`std::unique_ptr<duckdb::Expression>` to `std::unique_ptr<sirius::ast::node>`
is a breaking change to every `sirius::unwrap()` consumer. **34 files**
currently reference `sirius::expression`, `sirius::wrap`, or `sirius::unwrap`
under `src/` (excluding `legacy/`) — counted via grep on 2026-05-20.

Each unwrap caller that does
`sirius::unwrap(e)->Cast<duckdb::BoundXxxExpression>()` must already have a
Sirius-AST migration path by the time this PR lands. `sirius::join_condition`
(`src/include/expression/join_condition.hpp`) also holds wrapped expressions
in its `.left` / `.right` fields and must be migrated together.

### Phase 9–10 — Remove DuckDB-typed overloads & retire wrapper

**Risk:** Test migration. The two large executor-test files
(`test/cpp/expression_executor/test_gpu_expression_executor.cpp` = 2,030 LOC
and `.../test_gpu_expression_translator.cpp` = 1,823 LOC) currently build
test expressions out of `duckdb::Bound*Expression` instances. They must be
rewritten to build `sirius::ast::node` directly while preserving the 70/70
coverage count — any drop is a regression.

### Phase 11 — Non-DuckDB frontend smoke test ([#704](https://github.com/sirius-db/sirius/issues/704))

**Risk:** Authoritative acceptance criterion for #666 — a single failing
include-guard grep in the test body invalidates the whole milestone. Test
authoring is small but the grep-guard logic must be unambiguous.

### Cross-cutting: dual ecosystem during phases 4–9

While phases 4–9 are in flight, every change to expression handling must be
applied **twice** (once on the DuckDB-typed path, once on the Sirius-AST
path). The shim layer absorbs the duplication but does not eliminate it.
Avoid adding net-new functionality to the DuckDB-typed path during this
window unless absolutely required for correctness.

---

## Known Runtime Limits

### libcudf `int32_t` row-count limit (~2.1 billion rows per batch)

**Root cause:** cuDF uses `cudf::size_type` (= `int32_t`) for row IDs and
column lengths.

**Impact:** Any single GPU batch larger than `INT32_MAX` rows produces
undefined behavior in cuDF kernels. TPC-H scale ≥ 30,000 (3B+ `lineitem`
rows) exceeds this in unpartitioned operators.

**Current mitigation:**
- Sort partitioning splits oversized inputs (see `src/op/sort/` and
  `src/op/partition/`).
- The runtime fallback wrap in `src/sirius_extension.cpp:472` /
  `src/sirius_extension.cpp:508` (`Config::ENABLE_DUCKDB_FALLBACK`) catches
  any std::exception/throw and re-runs the query on DuckDB CPU.

**Gaps:**
- No explicit pre-flight row-count check at operator entry. A silently
  oversized batch will fail inside a cuDF kernel rather than being routed
  cleanly to CPU.
- `src/op/sirius_physical_grouped_aggregate.cpp`, `..._hash_join.cpp`,
  `..._nested_loop_join.cpp` do not currently validate row counts before
  invoking cuDF.
- *Unverified — needs human review:* whether the existing partitioning
  strategy covers `gpu_execution` (Super Sirius) on SF≥30000 inputs end-to-end.

### Unsupported logical operator types (CPU fallback via throw)

`src/planner/sirius_physical_plan_generator.cpp:135-309` throws
`duckdb::NotImplementedException` for 30 logical operator types. These are
**not GPU-supported** and fall back to DuckDB when
`Config::ENABLE_DUCKDB_FALLBACK = true` (default), or surface as a hard error
when fallback is disabled.

| Operator | Line | Notes |
|---|---|---|
| `LOGICAL_WINDOW` | 135 | TPC-H Q4/Q10/Q12/Q14/Q15 require RANK/ROW_NUMBER |
| `LOGICAL_UNNEST` | 139 | Nested type expansion |
| `LOGICAL_SAMPLE` | 146 | |
| `LOGICAL_COPY_TO_FILE` | 156 | |
| `LOGICAL_ANY_JOIN` | 163 | Non-equi join |
| `LOGICAL_ASOF_JOIN` | 167 | Time-series joins |
| `LOGICAL_CROSS_PRODUCT` | 174 | |
| `LOGICAL_POSITIONAL_JOIN` | 178 | |
| `LOGICAL_UNION` / `LOGICAL_EXCEPT` / `LOGICAL_INTERSECT` | 184 | Set ops |
| `LOGICAL_INSERT` / `_DELETE` / `_UPDATE` | 188 / 192 / 205 | Mutation |
| `LOGICAL_CREATE_TABLE` / `_INDEX` / `_SECRET` | 209 / 213 / 217 | DDL |
| `LOGICAL_EXPLAIN` | 221 | |
| `LOGICAL_DISTINCT` | 225 | Top-level DISTINCT (project-level DISTINCT works) |
| `LOGICAL_PREPARE` / `_EXECUTE` | 229 / 233 | Prepared statements |
| `LOGICAL_CREATE_VIEW` / `_SEQUENCE` / `_SCHEMA` / `_MACRO` / `_TYPE` | 241 | DDL |
| `LOGICAL_PRAGMA` / `_VACUUM` | 245 / 249 | |
| `LOGICAL_TRANSACTION` / `_ALTER` / `_DROP` / `_LOAD` / `_ATTACH` / `_DETACH` | 258 | "Simple" |
| `LOGICAL_RECURSIVE_CTE` | 262 | (Materialized CTE is supported.) |
| `LOGICAL_EXPORT` | 272 | |
| `LOGICAL_SET` / `_RESET` | 276 / 280 | |
| `LOGICAL_PIVOT` | 284 | |
| `LOGICAL_COPY_DATABASE` | 288 | |
| `LOGICAL_UPDATE_EXTENSIONS` | 292 | |
| `LOGICAL_EXTENSION_OPERATOR` | 296 | |
| `LOGICAL_JOIN` / `_DEPENDENT_JOIN` / `_INVALID` | 307 | "Unimplemented logical operator type!" |

**Fragility:** A new DuckDB version that adds a new `LogicalOperatorType`
enum value will reach the `default:` branch (line 309) and throw — the
fallback path then catches it, so the symptom is a silent CPU fallback for a
query category that used to be GPU-accelerated. Watch for DuckDB submodule
bumps (`duckdb` is pinned at `v1.5.2`, submodule SHA `8a585197...`).

### Unsupported type narrowing — silent data corruption

`src/include/cudf/cudf_utils.hpp:99-102` and `:107-110` narrow DuckDB's
`HUGEINT` (`INT128`) and `UHUGEINT` (`UINT128`) to cuDF `INT64`/`UINT64`
**without bounds checks**:

```cpp
case type_id::HUGEINT:
  // FIXME: unsafe 128→64-bit narrowing: DuckDB HUGEINT is INT128 but cuDF has no INT128.
  // Values outside INT64 range are silently corrupted. Matches legacy duckdb::GetCudfType().
  return cudf::data_type(cudf::type_id::INT64);
```

Same defect exists in the legacy mapper `duckdb::GetCudfType` at
`src/include/cudf/cudf_utils.hpp:240`. **Priority: HIGH (correctness).**
Tracked as inherited debt from the legacy mapper; no automated regression
guard.

**Fix path:** add runtime bounds check before the cuDF call, throw on
overflow so the fallback path catches it cleanly.

### Unsupported types — `LIST`, `SQLNULL`, `INVALID`

`src/include/cudf/cudf_utils.hpp:121-125` throws
`duckdb::InvalidInputException("sirius::get_cudf_type: Type %s cannot be
mapped to cuDF")` for `LIST`, `SQLNULL`, `INVALID`. `STRUCT` is mapped
(line 120) but only as a scalar — list types in projection or grouping go
to fallback.

**Implicit:** nested type expansion (`LOGICAL_UNNEST`) is rejected at the
plan-generator level (line 139), so list types only surface in scan output.

### Decimal AST disabled (cuDF upstream block)

`src/expression_executor/gpu_expression_executor.cpp:412-414` and
`src/expression_executor/specializations/gpu_execute_function.cpp:67-70`
disable cuDF AST lowering whenever the return type is `DECIMAL`:

```cpp
/// TODO: Fix when the following bug fix is in:
/// https://github.com/rapidsai/cudf/pull/21996
if (func_expr.return_type.id() == duckdb::LogicalTypeId::DECIMAL) { return 0; }
```

DECIMAL expressions fall through to the scalar (non-AST) path — slower but
correct. Blocked on upstream cuDF PR #21996.

### Memory tier behavior (cuCascade)

`src/downgrade/downgrade_executor.cpp` orchestrates GPU→HOST→DISK demotion
via cuCascade. Known sharp edges:

- `src/downgrade/downgrade_executor.cpp:172` calls
  `_data_repo_mgr.get_repositories()`. The 2026-04-24 audit flagged this as
  a "CRITICAL cuCascade API break"; on 2026-05-20 the call still exists, and
  HEAD compiles cleanly per STATE.md ("Build: clean, no warnings" as of
  2026-04-27). *Unverified — needs human review whether (a) cuCascade was
  rolled forward to restore `get_repositories()` (submodule SHA
  `9ceebaa7...` on `main-9-g9ceebaa`), or (b) the 2026-04-24 critical flag
  was over-stated. Recommend a fresh `make` to confirm.*
- `src/op/sirius_physical_operator.cpp:324` carries
  `// TODO: later on we will adjust to the new data repository interface in
  cuCascade` — interface migration is incomplete.
- `src/op/sirius_physical_result_collector.cpp:144` —
  `TODO: Find the closest memory space, not just any memory space, in HOST
  tier` — spill destination heuristic is suboptimal under partial HOST
  exhaustion.
- Downgrade testing is in `test/cpp/downgrade/` (`test_downgrade_executor.cpp`,
  `test_downgrade_lifecycle.cpp`, `test_downgrade_disk.cpp`) but coverage of
  predicate fairness, MPMC queue contention, and concurrent pipeline
  starvation is *unverified — needs human review*.

---

## Deferred Items (from `.planning/STATE.md`)

These are explicitly out-of-scope for v1.0 and should not be picked up
opportunistically inside Phase 7b PRs. Each will be re-evaluated in v2.

| Category | Item | Status | Rationale |
|---|---|---|---|
| Fallbacks | CASE in AST mode (v2 FB-01) | Deferred 2026-04-24 | Parity-with-7a milestone scope; CASE evaluates correctly today via scalar path. |
| Fallbacks | COALESCE in AST mode (v2 FB-02) | Deferred 2026-04-24 | Same as FB-01. |
| Fallbacks | ~~Route `regexp_replace` through AST~~ | **Retired** 2026-04-24 | Legacy-only `FallbackChecker` (`src/legacy/fallback.cpp:132`); regex already executes on GPU via cuDF in Super Sirius. |
| Extensibility | Pluggable function registry (v2 EXT-01) | Deferred 2026-04-24 | Closed `function_id` enum chosen for v1; registry can be layered on top later. |
| Extensibility | Stable Sirius AST API for other frontends (v2 EXT-02) | Deferred 2026-04-24 | Internal IR for v1; Phase 11 smoke test proves the surface, but no semver guarantee yet. |

---

## Build / Test Debt

### Worktree submodule init is manual

`CLAUDE.md` documents: after `git worktree add`, submodules are not
automatically initialized — must run
`git submodule update --init --recursive` by hand. There is no automation in
`Makefile` or pre-commit to enforce this; missing init produces confusing
build failures (missing headers from `cucascade/`, `duckdb/`, etc.).

**Submodules currently used (`git submodule status` 2026-05-20):**

```
bcfccda6 .claude/claude-tools (heads/main)
9ceebaa7 cucascade (heads/main-9-g9ceebaa)
8a585197 duckdb (v1.5.2)
2aea44ee duckdb-python (v1.4.1-511-g2aea44e)
245e2c29 extension-ci-tools (remotes/origin/sirius)
703de076 substrait (v1.2.0-17-g703de07)
ffc071e0 vcpkg (2026.01.16-492-gffc071e0c0)
```

### GPU compile memory pressure

`CLAUDE.md` warns:

```bash
CMAKE_BUILD_PARALLEL_LEVEL=$(nproc) make    # default
CMAKE_BUILD_PARALLEL_LEVEL=8 make           # if OOM
```

CUDA 13 + C++20 + `CMAKE_CUDA_SEPARABLE_COMPILATION ON` + 7 GPU
architectures (Turing, Ampere, Ada, Hopper, Blackwell) drives long compile
times and high per-job memory. No automation auto-tunes `parallel_level`
based on available RAM; CI machines hard-code it.

### `tpch-sirius.test` (SQLLogicTest) skipped during this milestone

Every `query` in `test/sql/tpch-sirius.test` calls `gpu_processing(...)` —
the legacy entry point. Because Phase 7b never touches legacy code, this
SQLLogicTest is reported as "SKIPPED" in plan-verify outputs (see
`.planning/STATE.md` Phase 3 verification block).

**Implication:** Super Sirius lacks an end-to-end TPC-H SQLLogicTest at the
default `gpu_execution` entry point. The C++ unit-test suite
(`sirius_unittest` — 79M+ assertions / 1,060 cases as of 2026-04-27)
substitutes for it; **but** there is no SQL-level regression guard for
Super Sirius. Adding one is a candidate post-milestone item.

### Pre-commit discipline

`pre-commit` (clang-format, clang-tidy with `WarningsAsErrors: '*'`,
codespell, cmake-format, black) is mandatory; PROJECT.md states "No hooks,
no `--no-verify`". Skipping is forbidden per `<<EOF >> commit policy>>`.
Watch for `clang-tidy` regressions on cucascade or duckdb submodule bumps —
warnings-as-errors means a transitive header change can fail an unrelated
PR.

### CI signal scope

`.github/workflows/` (not enumerated here) is *unverified — needs human
review* for: (a) whether Super Sirius SF1 timing is gated at PR-merge time
or only on `merge_group`, (b) whether `-DENABLE_LEGACY_SIRIUS=ON` builds
both compile and run unit tests, (c) which RAPIDS cuDF version is the CI
default (stable vs nightly).

---

## Documentation / Process Debt

### Stray `.planning/` commits before `commit_docs=false` was configured

Two planning commits landed in branch history before `.planning/config.json`
set `"commit_docs": false`:

- `78eedcb7 docs: initialize project` (was `a1ae8906` in the original
  un-rebased history)
- `b6144eb7 docs: map existing codebase` (was `90151e9f` originally)

They are present on the active branch `sirius_expression_framework`. When
`/gsd-pr-branch` extracts per-phase PR branches for upstream, these planning
commits get filtered out — but they remain in this worktree's local history.
User to decide whether to squash/untrack later. See `.planning/STATE.md`
Blockers/Concerns.

**Subsequent planning commits** (`d823e9cf`, `147ab026`, `e8946f59`,
`3ebe110e`) were intentional `.planning/` updates committed under the
"sync planning state to git" workflow — these are expected, not strays.

### Codebase docs are auto-aged

This `CONCERNS.md` is dated **2026-05-20** (current refresh). Sibling docs
in `.planning/codebase/` (`ARCHITECTURE.md`, `CONVENTIONS.md`,
`INTEGRATIONS.md`, `STACK.md`, `STRUCTURE.md`, `TESTING.md`) carry their own
"Analysis Date" headers. Drift accumulates between refreshes — re-run
`/gsd-map-codebase` periodically (or after major upstream merges) to keep
them in sync. Last full refresh of those siblings was 2026-04-24 per the
commit log; STACK/STRUCTURE/TESTING/CONVENTIONS may carry pre-PR-#692/#693
references.

### Super Sirius docs

`docs/super-sirius/README.md` carries
`<!-- last-updated-commit: d6baaedc0d2bb07b27a00c65135513f3c23f0b37 -->` —
suggests a manual stamp, not automated. The `update-docs` skill exists
(`.claude/skills/update-docs/SKILL.md`) but is contributor-driven. Watch
for drift after each merged Super Sirius PR.

### Phase 7b roadmap is private

The 11-phase plan and per-phase risk surface live in `.planning/ROADMAP.md`,
`.planning/PROJECT.md`, and `.planning/STATE.md` — not in `docs/` or
upstream. Upstream visibility is via the 11 GitHub sub-issues under
[#666](https://github.com/sirius-db/sirius/issues/666). Contributors who
don't read `.planning/` may not see the phase ordering or the dual-path
window.

---

## Fragile Areas

Contributors should be cautious when editing these files. Each is a hot
path with non-obvious invariants.

### `src/include/op/sirius_physical_operator.hpp:406` — `Cast<TARGET>()`

```cpp
template <class TARGET>
TARGET& Cast()
{
  // TODO(amin) this is buggy code
  if (TARGET::TYPE != SiriusPhysicalOperatorType::INVALID && type != TARGET::TYPE) {
    throw internal_exception(
      "Failed to cast physical operator to type - physical operator type mismatch");
  }
  return reinterpret_cast<TARGET&>(*this);
}
```

- Runtime-only type check, then `reinterpret_cast`. The "buggy code" comment
  has not been resolved.
- A subclass with `TARGET::TYPE == INVALID` bypasses the check entirely and
  reinterprets blindly.
- **Don't add new operator types without a corresponding `SiriusPhysicalOperatorType`
  enum entry** in `src/op/sirius_physical_operator_type.cpp`.

### `src/include/op/sirius_physical_operator.hpp:483, 490` — unimplemented virtuals

```cpp
virtual bool can_create_more_tasks() const
{
  // WSM TODO implement this
  throw std::runtime_error("can_create_more_tasks not implemented for operator " + get_name());
}
```

Same pattern at `has_processed_all_tasks()` (around line 490 — *unverified —
look near the throw above*). If a new operator subclass forgets to override
these, the pipeline executor throws at runtime in error-recovery / drain
paths.

### `src/expression_executor/gpu_expression_translator.cpp` (725 LOC)

- Still consumes `duckdb::Expression` directly (Phase 7 will flip).
- Owns cuDF scalar lifetimes — see the in-file comment near line 484:
  `// TODO: Expand type support as needed. See gpu_execute_constant.cpp.`
- During Phase 7 flip, **the existing 9
  `add_expression(duckdb::Bound*Expression const&, ...)` overloads must be
  deleted atomically** with the migration of all 17 plan-builder call sites
  in `src/planner/`. A half-finished flip will not compile.

### `src/expression_executor/gpu_expression_executor.cpp:412` — DECIMAL gate

The early-return-zero (skip AST) for DECIMAL outputs is structural — every
new function-handling path must respect it until cuDF PR #21996 lands. Easy
to miss when adding a new function specialization.

### `src/include/expression/expression.hpp` + `expression_internal.hpp` — PIMPL

The opaque wrapper exists *exactly* so that operator/plan-builder/executor
headers do not include `duckdb/planner/expression/*`. Adding a new include
of `duckdb::Bound*Expression` in a header that uses `sirius::expression` is
a regression — re-route through `sirius::unwrap()` in the `.cpp` only.

### `src/op/sirius_physical_table_scan.cpp`, `_parquet_scan.cpp`, `_iceberg_scan.cpp`, `_duckdb_scan.cpp` — scan parameters

Scan operators accept `duckdb::Value` in parameter lists (table parameters,
hive partition values, filter pushdown constants). PROJECT.md (Out of Scope)
explicitly defers their migration to **post-Phase-7**. Don't try to migrate
them inside this milestone.

### `src/expression_executor/specializations/gpu_execute_function.cpp` — function dispatch

Phase 3 (merged) replaced 27 `*_FUNC_STR` macros with the closed
`sirius::function_id` enum. The active surface is the
`switch (function_id)` block; adding a new function means:

1. Add an entry to `enum class function_id` in
   `src/include/expression/function_id.hpp`.
2. Add a name mapping in `src/expression/function_id.cpp` (round-trip).
3. Add a case in `gpu_execute_function.cpp`.
4. Update `supported_ast_functions` in
   `src/include/expression_executor/ast_supported_types.hpp:39` if the
   function is safe to lower to cuDF AST.

Skipping step 4 produces a *silent* non-AST path — correct, but slow.

### `src/planner/sirius_plan_comparison_join.cpp:299` — PWMJ limitation

```cpp
// TODO: Extend PWMJ to handle all comparisons and projection maps
```

Piecewise-merge join doesn't handle every comparison; non-trivial join
conditions go to nested-loop join or fallback.

### `src/creator/task_creator.cpp:168, 349` — port + hint heuristics

```cpp
// WSM TODO: how do we handle other ports that are not default?
// TODO(amin) : do this based on the operator hint
```

Task creation heuristics are partial. Touching scan or partition operator
port wiring may expose these.

### `src/op/sirius_physical_partition.cpp:121` — partition key extraction

```cpp
// WSM TODO: this is the original code for getting the partition keys from
// the grouped aggregate
```

Carried-forward code from the legacy partition implementation; works for
TPC-H but not exercised by every operator path.

### `src/include/config.hpp:38, 69` — hardcoded heuristics

Task sizing (`DEFAULT_SCAN_TASK_BATCH_SIZE`, `DEFAULT_SCAN_TASK_VARCHAR_SIZE`)
and the expression executor strategy (`EXPRESSION_EXECUTOR_STRATEGY`) are
config-driven but **not adaptive** per-call. Different GPUs / dataset shapes
need different values; the TODOs in this header acknowledge it.

---

## Test Coverage Gaps

### Expression boundary values

- `HUGEINT` literals > `INT64_MAX` (silent narrowing — see Known Runtime Limits).
- `UHUGEINT` literals > `UINT64_MAX`.
- CAST crossing type bounds (e.g. `DOUBLE → BIGINT` with `NaN`, `±Inf`).
- `NULL` propagation through `CASE`, `BETWEEN`, function arguments.

Add tests to `test/cpp/expression/test_ast_scaffold.cpp` (or successor
files after Phase 4) once `sirius::value` round-trip is the path under test.

### Downgrade executor edge cases

`test/cpp/downgrade/` covers basics, but the following are *unverified —
needs human review*:

- Predicate early-stop fairness when many concurrent requesters share the
  same MPMC queue.
- Pool defragmentation under fragmentation pressure.
- DISK tier fallback latency under near-full HOST tier.
- Re-entry after `_pool->reserve()` is unblocked by a previous candidate
  finishing (see `src/downgrade/downgrade_executor.cpp:189-191`).

### Non-DuckDB frontend (proof test)

The whole point of Phase 11 — currently zero coverage. Until Phase 11 lands,
the claim "Sirius AST is frontend-independent" is *unverified*.

### Operator task lifecycle on unimplemented virtuals

Pipelines that hit `can_create_more_tasks()` or `has_processed_all_tasks()`
on operators that haven't overridden them currently throw at runtime — no
static guard. Add a CMake/clang-tidy rule, or override these methods in the
base class with a sensible default. *Recommended action.*

---

## Dependencies at Risk

### cuDF (RAPIDS)

- **`int32_t` row IDs** (see Known Runtime Limits).
- **PR rapidsai/cudf#21996** — DECIMAL AST intermediate support — blocks
  re-enabling AST for decimal expressions. Status: *unverified — needs
  human review* (last checked 2026-04-24).
- Version handling: `src/include/cudf/cudf_utils.hpp:19-43` branches on
  `CUDF_VERSION_NUM` for `>2504` and `>=2604`. Pixi env locks the cuDF
  version; submodule SHA `9ceebaa7...` (cucascade) tracks the pinned
  combination.

### cuCascade

- API drift around `shared_data_repository_manager::get_repositories()`
  (see Memory tier behavior above). Currently builds clean; needs a fresh
  full `make` to confirm. Submodule SHA `9ceebaa7...` on `main-9-g9ceebaa`.

### DuckDB

- Pinned at `v1.5.2` (submodule SHA `8a585197...`).
- `gpu_expression_translator.cpp` and `sirius_physical_plan_generator.cpp`
  switch on `duckdb::LogicalOperatorType` / `duckdb::ExpressionClass` /
  `duckdb::LogicalTypeId`. A DuckDB minor bump can silently add new enum
  values that hit the `default:` arm and trigger CPU fallback — not a
  correctness issue, but a stealth performance regression.
- Compile-time: every DuckDB header included by Sirius transitively brings
  in C++20-heavy templates; this is a meaningful share of the long compile
  times noted above.

### substrait

- Pinned at `v1.2.0-17`. PROJECT.md "Out of Scope" explicitly defers
  Substrait interop — but the submodule is still present and compiled into
  the extension. *Unverified — needs human review whether substrait is
  actively linked or can be dropped.*

---

## Summary of Highest-Priority Concerns (2026-05-20)

| Priority | Concern | Location | Action |
|---|---|---|---|
| **HIGH** | HUGEINT/UHUGEINT silent narrowing | `src/include/cudf/cudf_utils.hpp:99-110, 240-247` | Add bounds check + throw → fallback. |
| **HIGH** | Phase 7b dual-path window risk | phases 5–9 | Land each phase as a single revertable PR; never merge two half-flips together. |
| **MEDIUM** | cuCascade `get_repositories()` status | `src/downgrade/downgrade_executor.cpp:172` | Fresh `make` + run `test/cpp/downgrade/*` to confirm. |
| **MEDIUM** | No SQL-level regression guard for Super Sirius | `test/sql/` (only legacy `tpch-sirius.test`) | Author a `gpu_execution`-based SQLLogicTest post-Phase-11. |
| **MEDIUM** | `sirius_physical_operator::Cast<>` reinterpret_cast safety | `src/include/op/sirius_physical_operator.hpp:406` | Replace with type-safe visitor or document failure modes. |
| **MEDIUM** | Unimplemented `can_create_more_tasks()` virtual throws | `src/include/op/sirius_physical_operator.hpp:483` | Default to `false` or audit overrides per operator. |
| **LOW** | Submodule init manual on worktrees | `CLAUDE.md` (Git Worktrees section) | Add a `make submodules` helper or pre-commit guard. |
| **LOW** | Stray `.planning/` commits in branch history | `78eedcb7`, `b6144eb7` | Decide squash/untrack at PR-extract time. |
| **LOW** | DECIMAL AST gate (upstream-blocked) | `gpu_expression_executor.cpp:412`, `gpu_execute_function.cpp:67-70` | Track cuDF PR #21996; remove gate once merged. |

---

*Concerns refresh: 2026-05-20*
*Supersedes 2026-04-24 audit. Next refresh recommended after Phase 5 (dual-path executor) lands.*
