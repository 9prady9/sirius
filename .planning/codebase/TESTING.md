# Testing Patterns

**Analysis Date:** 2026-05-20

Sirius runs three test systems in parallel: a Catch2-based C++ unit-test binary
(`sirius_unittest`), DuckDB's SQLLogicTest harness for end-to-end SQL, and a
Python TPC-H performance harness. During the v1.0 milestone the **only**
mandatory regression gate is the `sirius_unittest` binary, including its 70/70
expression-executor suite. SQLLogicTests are skipped because they exercise the
frozen legacy `gpu_processing` path.

## Test Frameworks

**C++ unit tests:** Catch2 v2 (vendored at `duckdb/third_party/catch/catch.hpp`,
picked up via `target_include_directories(sirius_unittest ...)` in
`CMakeLists.txt:466`). Single-binary runner with the registered
`shared_env_listener` for environment management.

**SQL logic tests:** DuckDB's SQLLogicTest harness (`build/release/test/unittest
--test-dir . test/sql/*.test`). Currently skipped — `make test` is a stub that
echoes a message and does not invoke the harness (see `Makefile:96-112`).

**Performance / regression harness:** Python + bash under
`test/tpch_performance/` (described below).

## Build and Run Commands

```bash
# Build sirius_extension + sirius_unittest (release)
CMAKE_BUILD_PARALLEL_LEVEL=$(nproc) make

# Run the full unit-test binary
build/release/extension/sirius/test/cpp/sirius_unittest

# Run a tag subset (Catch2 syntax — brackets are required)
build/release/extension/sirius/test/cpp/sirius_unittest "[ast_scaffold]"
build/release/extension/sirius/test/cpp/sirius_unittest "[ast_value]"
build/release/extension/sirius/test/cpp/sirius_unittest "[ast_function_id]"
build/release/extension/sirius/test/cpp/sirius_unittest "[expression_executor]"
build/release/extension/sirius/test/cpp/sirius_unittest "[physical_filter]"

# Run a specific test case by substring match on the case name
build/release/extension/sirius/test/cpp/sirius_unittest \
  "ast_function_id - add round-trips through name mappers"

# Abort on first failure (CI uses this)
build/release/extension/sirius/test/cpp/sirius_unittest --abort

# Debug build variant
CMAKE_BUILD_PARALLEL_LEVEL=$(nproc) make debug
build/debug/extension/sirius/test/cpp/sirius_unittest

# Legacy-enabled build (adds the [cpu_cache] suite and the legacy gpu_processing
# code path; required to exercise tpch-sirius.test)
CMAKE_BUILD_PARALLEL_LEVEL=$(nproc) make legacy-release
build/legacy-release/extension/sirius/test/cpp/sirius_unittest "[cpu_cache]"
```

**`make test` is a stub during the v1.0 milestone.** It prints:

> SQL logic tests use the legacy gpu_processing path and are skipped by default.
> Run C++ unit tests with: ./build/release/extension/sirius/test/cpp/sirius_unittest

(see `Makefile:96-112`). To actually run the SQL logic tests you must build with
`make legacy-release` and invoke `duckdb/build/release/test/unittest` manually.

## Test File Organization

```
test/cpp/
├── config/                            # configuration loader tests
├── creator/                           # task creator
├── data/                              # data batch / conversion tests
├── debug/
├── downgrade/                         # GPU→HOST→DISK downgrade pipeline
├── exec/                              # bounded thread pool, MPMC/MPSC channels
├── expression/                        # AST nodes, sirius::value, function_id
│   ├── test_ast_scaffold.cpp
│   ├── test_expression.cpp
│   ├── test_function_id.cpp           # Phase 3
│   ├── test_join_condition.cpp
│   └── test_value.cpp                 # Phase 2
├── expression_executor/               # the 70/70 regression gate
│   ├── test_gpu_expression_executor.cpp
│   └── test_gpu_expression_translator.cpp
├── helper/
├── integration/                       # full-pipeline GPU tests
│   ├── data/duckdb/integration.duckdb
│   ├── integration.yaml
│   ├── test_gpu_execution_multi_format.cpp
│   ├── test_gpu_execution_tpch.cpp
│   ├── test_tpcds_plan_translation.cpp
│   └── test_transparent_execution.cpp
├── io/                                # URI parser, datasource factory, S3 SigV4
├── memory/
├── memory_management/                 # legacy-only ([cpu_cache])
├── operator/                          # Super Sirius physical operators
│   ├── aggregate/
│   ├── operator_test_utils.hpp
│   ├── operator_type_traits.hpp
│   ├── test_physical_filter.cpp
│   ├── test_physical_projection.cpp
│   ├── test_physical_top_n.cpp
│   ├── test_sirius_dynamic_filter.cpp
│   └── ...
├── parallel/
├── pipeline/                          # task scheduler, pipeline executor, plan printer
├── planner/                           # plan-translation tests
├── scan/                              # parquet/iceberg scan, GPU decoders
│   └── memory.yaml                    # config for [shared_context] tests
├── utils/                             # shared test environment + helpers
│   ├── sirius_test_env.hpp/.cpp
│   ├── utils.hpp/.cpp
│   ├── data_utils.hpp
│   ├── test_validation_utility.hpp
│   └── transparent_execution_test_utils.hpp
└── unittest.cpp                       # Catch2 runner + shared_env_listener
```

**Test source list:** `CMakeLists.txt:371-449` (between `# cmake-format: off` /
`# cmake-format: on` fences). Adding a new test means appending the file path
to `TEST_SOURCES`.

**Legacy-only test sources:** `src/legacy/CMakeLists.txt:106-108` registers
`test/cpp/memory_management/test_cpu_cache.cpp` via `SIRIUS_LEGACY_TEST_SOURCES`,
which gets appended to `TEST_SOURCES` only when `ENABLE_LEGACY_SIRIUS=ON`.

**Naming:**
- Test file: `test_<component>.cpp` (e.g., `test_function_id.cpp`,
  `test_physical_filter.cpp`).
- Test case: `TEST_CASE("descriptive sentence about behavior", "[tag1][tag2]")`.
- Parameterized: `TEMPLATE_TEST_CASE(name, tags, T1, T2, ...)`.
- Sections inside a case: `SECTION("subsection description")`.

## Test Structure

```cpp
#include "catch.hpp"
#include "expression/function_id.hpp"

#include <cstdint>
#include <optional>
#include <string_view>

using sirius::from_duckdb_function_name;
using sirius::function_id;

// Compile-time invariants up top (locks ABI cardinality, sizeof, etc.)
static_assert(sizeof(function_id) == 2, "uint16_t-backed ABI");

TEST_CASE("ast_function_id - add round-trips through name mappers", "[ast_function_id]")
{
  auto const name = to_duckdb_function_name(function_id::add);
  REQUIRE(name == "+");
  auto const id = from_duckdb_function_name(name);
  REQUIRE(id.has_value());
  REQUIRE(*id == function_id::add);
}
```

**Patterns:**
- Compile-time invariants (`static_assert`) appear at file scope, before any
  `TEST_CASE`. Used heavily in
  `test/cpp/expression/test_ast_scaffold.cpp`,
  `test/cpp/expression/test_value.cpp`, and
  `test/cpp/expression/test_function_id.cpp` to lock ABI cardinality, backing
  type widths, and move-only contracts.
- Arrange-Act-Assert ordering with `REQUIRE` for hard failures and `CHECK` for
  soft checks that should continue.
- Helpers in anonymous namespaces or in `<component>_test_utils.hpp` headers
  (e.g., `test/cpp/operator/operator_test_utils.hpp`).
- Shared GPU memory configuration via
  `sirius::test::operator_utils::initialize_memory_manager(n_gpus)`.

## Test Tags

Tags select a subset of cases at the CLI: `sirius_unittest "[tag_name]"`. Tags
also drive the `shared_env_listener` in `test/cpp/unittest.cpp`.

**Environment-routing tags** (handled by `shared_env_listener`):
- `[shared_context]` — needs the shared `g_shared_env` (one DuckDB instance,
  one SiriusContext, scan-oriented config at `test/cpp/scan/memory.yaml`).
- `[integration]` — needs the shared `g_integration_env` (full-pipeline GPU
  execution, integration config at `test/cpp/integration/integration.yaml`).
- Tests without either tag run in isolation; both shared envs are paused.

**Active per-component tags** (representative):

| Tag | Area | File(s) |
|-----|------|---------|
| `[ast_scaffold]` | AST node scaffold (Phase 1) | `test/cpp/expression/test_ast_scaffold.cpp` |
| `[ast_value]` | `sirius::value` payload + DuckDB mappers (Phase 2) | `test/cpp/expression/test_value.cpp` |
| `[ast_function_id]` | `sirius::function_id` enum + name mappers (Phase 3) | `test/cpp/expression/test_function_id.cpp` |
| `[expression]` | PIMPL wrapper + `join_condition` | `test/cpp/expression/test_expression.cpp`, `test_join_condition.cpp` |
| `[expression_executor]` | **70/70 regression gate** | `test/cpp/expression_executor/test_gpu_expression_executor.cpp`, `test_gpu_expression_translator.cpp` |
| `[physical_filter]` | FILTER operator | `test/cpp/operator/test_physical_filter.cpp` |
| `[physical_projection]` | PROJECTION operator | `test/cpp/operator/test_physical_projection.cpp` |
| `[physical_top_n]`, `[physical_top_n_merge]` | TOP-N | `test/cpp/operator/test_physical_top_n.cpp` |
| `[physical_order]` | ORDER BY | `test/cpp/operator/test_physical_order.cpp` |
| `[physical_merge_sort]` | merge sort | `test/cpp/operator/test_physical_merge_sort.cpp` |
| `[physical_partition]` | partition operator | `test/cpp/operator/test_physical_partition.cpp` |
| `[physical_concat]` | concat | `test/cpp/operator/test_physical_concat.cpp` |
| `[physical_limit]` | LIMIT | `test/cpp/operator/test_physical_limit.cpp` |
| `[physical_mark_join]` | mark join | `test/cpp/operator/test_physical_mark_join.cpp` |
| `[physical_table_scan]` | table scan | `test/cpp/operator/test_physical_table_scan.cpp` |
| `[physical_grouped_aggregate]`, `[physical_ungrouped_aggregate]` | aggregations | `test/cpp/operator/aggregate/...`, `test_physical_ungrouped_aggregate.cpp` |
| `[dynamic_filter]` | runtime filter set | `test/cpp/operator/test_sirius_dynamic_filter.cpp` |
| `[task_scheduler]`, `[pipeline_queue]` | pipeline runtime | `test/cpp/pipeline/test_task_scheduler.cpp` |
| `[gpu_pipeline_executor]` | pipeline executor | `test/cpp/pipeline/test_gpu_pipeline_executor.cpp` |
| `[gpu_pipeline_disk]` | DISK↔GPU readback | `test/cpp/pipeline/test_gpu_pipeline_disk_readback.cpp` |
| `[plan_printer]` | plan visualization | `test/cpp/pipeline/test_plan_printer.cpp` |
| `[task_creator]`, `[task_executor]` | task creation / executor | `test/cpp/creator/test_task_creator.cpp`, `test/cpp/parallel/test_task_executor.cpp` |
| `[downgrade_disk]`, `[downgrade_executor]`, `[downgrade_lifecycle]` | downgrade pipeline | `test/cpp/downgrade/*.cpp` |
| `[uri_parser]`, `[datasource_factory]`, `[parquet_schema_mapping]` | I/O | `test/cpp/io/*.cpp`, `test/cpp/scan/test_parquet_schema_mapping.cpp` |
| `[object_store_config]` | object-store config | `test/cpp/config/test_object_store_config.cpp` |
| `[convertible_data_batch]`, `[convertible_gpu_pipeline_task]`, `[host_parquet_representation]` | data batch / conversion | `test/cpp/data/*.cpp` |

**Legacy-only tag** (requires `make legacy-release`):
- `[cpu_cache]` — `test/cpp/memory_management/test_cpu_cache.cpp`. Not built in
  the default release.

Run the canonical regression gate:
```bash
build/release/extension/sirius/test/cpp/sirius_unittest "[expression_executor]"
```

Run all AST-related unit tests:
```bash
build/release/extension/sirius/test/cpp/sirius_unittest \
  "[ast_scaffold],[ast_value],[ast_function_id]"
```

## Shared Test Environments

`test/cpp/unittest.cpp` registers a Catch2 `TestEventListener` named
`shared_env_listener` (lines 45-85). It dispatches across two globally-managed
`sirius::test::shared_test_env` instances:

- `sirius::test::g_shared_env` — created with
  `test/cpp/scan/memory.yaml` for `[shared_context]` tests.
- `sirius::test::g_integration_env` — created with
  `test/cpp/integration/integration.yaml` for `[integration]` tests.

**Transition logic** (`testCaseStarting`):
- Classify each test by tag: `SHARED`, `INTEGRATION`, or `NONE`.
- Pause whichever environment is currently active but doesn't match the
  classification (only one may be active at a time — each owns the extension
  lock).
- Resume the matching environment if needed.

**Lifecycle** (`shared_test_env::pause()` / `resume()` in
`test/cpp/utils/sirius_test_env.hpp`):
- `pause()` destroys the `duckdb::DuckDB` instance.
- `resume()` re-creates it via `create_db()` — re-triggering the extension
  callback and re-creating the `SiriusContext`.
- Both environments start paused; the first matching test resumes them.

**Why the listener exists:** consecutive `[shared_context]` tests (or
consecutive `[integration]` tests) reuse the same DuckDB instance and
SiriusContext, avoiding the cost of repeated extension load and GPU context
creation. Switching tag families pays the pause/resume cost once.

## Mocking

Sirius does not use a mocking framework. Test patterns:

- **Real DuckDB expression trees:** construct via
  `duckdb::make_uniq<BoundComparisonExpression>(...)` etc., wrap with
  `sirius::wrap(...)`.
- **Real GPU operators:** the operator-test pattern is "initialize memory
  manager → build batch → call `operator.execute(input, stream)` → cast and
  inspect outputs". See `test/cpp/operator/test_physical_filter.cpp` for the
  canonical shape.
- **Real cuDF data:** `cucascade::gpu_table_representation` + `cudf::table` —
  no mocks. The test memory manager (`initialize_memory_manager`) configures a
  512MB GPU space and 1GB host space (`test/cpp/operator/operator_test_utils.hpp`).
- **Hand-built AST nodes (Phase 1+ tests):** in `test_ast_scaffold.cpp`,
  `test_value.cpp`, and `test_function_id.cpp` the tests construct
  `sirius::ast::*` nodes and `sirius::value` instances by hand — no DuckDB
  dependency at all (modulo the boundary round-trip checks).

## Fixtures and Factories

**Memory manager + GPU space** (`test/cpp/operator/operator_test_utils.hpp`):
```cpp
auto memory_manager = sirius::test::operator_utils::initialize_memory_manager(n_gpus);
auto* space = memory_manager->get_memory_space(cucascade::memory::Tier::GPU, 0);
```

**Typed batch factory:**
```cpp
auto batch = sirius::test::operator_utils::make_batch<int64_t>(*space, input_vals);
auto batch = sirius::test::operator_utils::make_two_column_batch<int64_t, TestType>(
  *space, filter_vals, data_vals, Traits::cudf_type, std::nullopt);
```

**Host-side verification:**
```cpp
auto host_vals = copy_column_to_host<int64_t>(output_table.view().column(0));
REQUIRE(host_vals == expected_vals);
```

**Test fixtures for integration tests** (`test_gpu_execution_tpch.cpp`):
- `GPUExecutionFixtureBase` holds an owned or shared `duckdb::DuckDB` +
  `duckdb::Connection`. When `g_integration_env` is active, the connection is
  borrowed from the env (`g_integration_env->make_connection()`); otherwise the
  fixture creates an isolated DuckDB.

**Test data and configs:**
- Integration DuckDB seed: `test/cpp/integration/data/duckdb/integration.duckdb`
  (excluded from the pre-commit `check-added-large-files` hook).
- Integration env config: `test/cpp/integration/integration.yaml`.
- Scan env config: `test/cpp/scan/memory.yaml`.
- TPC-H data for the performance harness: generated on demand via
  `test/tpch_performance/generate_test_data.py`.

## Common Patterns

**GPU streams.** Operator tests pass `cudf::get_default_stream()` for
synchronization; cuDF/RMM ops are implicitly serialized on that stream:
```cpp
auto outputs = filter.execute(pipelineable_operator_data(inputs),
                              cudf::get_default_stream());
```

**Exception expectations.** Catch2's `REQUIRE_THROWS_AS` /
`REQUIRE_THROWS_WITH`:
```cpp
REQUIRE_THROWS_AS(sirius::from_duckdb(unsupported_value, unsupported_type),
                  sirius::not_implemented_exception);
```

**Templated parameterization.** For the operator suite, `TEMPLATE_TEST_CASE`
sweeps numeric types:
```cpp
TEMPLATE_TEST_CASE("sirius_physical_filter applies comparison",
                   "[physical_filter]",
                   int32_t, int64_t, float, double, decimal64_tag, ...)
{
  using Traits = gpu_type_traits<TestType>;
  // ...
}
```

**Strategy parameterization.** The expression executor's 70-test suite uses
`TEMPLATE_TEST_CASE` over three strategy tags (`mat_strategy`,
`ast_interpret_strategy`, `ast_jit_strategy`) so every case runs under
MATERIALIZE, AST_INTERPRET, and AST_JIT:
```cpp
TEMPLATE_TEST_CASE("select BETWEEN", "[expression_executor]",
                   mat_strategy, ast_interpret_strategy, ast_jit_strategy)
{ ... }
```

## Coverage

No coverage enforcement is configured in CMake; the gate is the test pass/fail
plus the TPC-H SF1 perf snapshot in CI. Test counts at the last known-good
verification (2026-04-27 — see `.planning/STATE.md`):
- `[ast_function_id]`: 88 assertions / 31 cases
- `[ast_value]`: 84 assertions / 27 cases
- `[ast_scaffold]`: 44 assertions / 13 cases
- Full `sirius_unittest`: ~79M assertions across 1060 cases

## CI Gates

Two GitHub Actions workflows guard `dev` and every PR
(`.github/workflows/check.yml`, `.github/workflows/test.yml`).

**`check.yml` (Check):**
- `lint` — runs `pre-commit run --show-diff-on-failure --all-files` on
  ubuntu-24.04.
- `build` — matrix build on x64 + arm64 across gcc/clang, release/debug, and a
  vcpkg variant. Includes a `legacy` variant that builds with
  `-DENABLE_LEGACY_SIRIUS=ON`.
- Failure of any matrix cell fails the `check` aggregator job.

**`test.yml` (Test):**
- `build-cuda-13` / `build-cuda-12` — pixi-driven builds on
  self-hosted runners (`cuda-13` + `cpu-xl`, `cuda-12` + `cpu-xl`).
- `test-cuda-13` / `test-cuda-12` — on T4 GPU runners, with
  `SIRIUS_LOG_LEVEL=trace`, runs:
  ```bash
  timeout 45m ./build/release/extension/sirius/test/cpp/sirius_unittest --abort
  ```
- After the unit test, both jobs prepare TPC-H SF1 data and run
  `test/tpch_performance/benchmark_and_validate.sh 1`. Any erroring TPC-H
  query fails the merge-queue job.
- Unit-test logs are uploaded as artifacts from
  `build/release/extension/sirius/test/cpp/log/`.

**Phase 7b additional constraint** (from `.planning/PROJECT.md`):
- `sirius_unittest` must pass on every commit on both default and
  `-DENABLE_LEGACY_SIRIUS=ON` builds.
- No TPC-H SF1 regression across `materialize`, `ast_interpret`, and `ast_jit`
  strategies.

## SQLLogicTest (SKIPPED for v1.0)

`test/sql/` contains three SQLLogicTest files:

| File | Status |
|------|--------|
| `test/sql/tpch-sirius.test` | **SKIPPED.** Uses `call gpu_processing("...")` — the legacy entry point. Phase 7b never touches the legacy path. |
| `test/sql/clickbench-sirius.test` | **SKIPPED.** Same legacy `gpu_processing` constraint. |
| `test/sql/bugfix.test` | Legacy bugfixes; not run in default CI. |

To run them locally you must build with `make legacy-release` and invoke the
DuckDB unittest binary directly:
```bash
make legacy-release
build/legacy-release/test/unittest --test-dir . test/sql/tpch-sirius.test
```

The mainline `make test` target is intentionally a stub that prints a reminder
to use `sirius_unittest` instead.

## Performance Harness

Located at `test/tpch_performance/` — a self-contained pixi environment plus
shell drivers. Key entry points:

| File | Purpose |
|------|---------|
| `generate_test_data.py` | Generates TPC-H data at a given scale factor. |
| `generate_tpch_data.sh`, `generate_tpch_dataset_legacy.sh` | Wrap data generation. |
| `performance_test.py` | Runs every TPC-H query against Super Sirius and DuckDB and reports timings. |
| `benchmark_and_validate.sh` | CI entry — runs the benchmark and writes `comparison.txt` + `validation.csv` into a `runs/<timestamp>/` directory. |
| `run_tpch_parquet.sh`, `run_tpch_parquet_duckdb.sh`, `run_tpch_legacy.sh`, `run_iceberg_benchmark.sh` | Parquet / Iceberg / legacy variants. |
| `nsys_report.sh`, `nsys_hotspots.sh`, `nsys_analyze.sh`, `nsys_compare.sh`, `profile_tpch_nsys.sh` | Nsight Systems profiling drivers. |
| `compare_results.py` | Cross-run timing comparison. |
| `tpch_queries/` | TPC-H SQL templates. |
| `scan_cache_levels.yaml`, `pixi.toml`, `pixi.lock` | Environment configuration. |

Build the duckdb-python wheel once before using the harness:
```bash
pixi run -e duckdb-python build-duckdb-python
```

Then run a benchmark:
```bash
cd test/tpch_performance
./benchmark_and_validate.sh 1          # SF1
```

## Log Directories

- **Unit tests (`sirius_unittest`):**
  `build/release/extension/sirius/test/cpp/log/` (uploaded as a CI artifact on
  every test run). Configured by the `SIRIUS_UNITTEST_LOG_DIR` compile
  definition in `CMakeLists.txt:489`.
- **Engine (`gpu_execution`, embedded extension):** controlled by
  `SIRIUS_LOG_DIR` env var; defaults to `${CMAKE_BINARY_DIR}/log` via the
  `SIRIUS_DEFAULT_LOG_DIR` compile definition.
- **Log level:** `SIRIUS_LOG_LEVEL` env var
  (`trace | debug | info | warn | error`). CI sets `trace`; default is `info`.
- Both env vars are read in `src/sirius_context.cpp:707-709`.

## Adding a New Test

1. Create `test/cpp/<area>/test_<feature>.cpp` mirroring an existing layout
   (e.g., copy the shape of `test/cpp/expression/test_function_id.cpp` for a
   non-GPU type test, or `test/cpp/operator/test_physical_filter.cpp` for a
   GPU operator test).
2. Add the Apache 2.0 license header (copy from any sibling).
3. `#include "catch.hpp"` first, then component headers, then `<cstdint>` /
   other STL.
4. Choose tags:
   - No DuckDB, no GPU → use `[ast_scaffold]`-style tags
     (`[ast_function_id]`, `[ast_value]`, or a new pure-type tag).
   - Scan/operator unit needing a shared `SiriusContext` → add `[shared_context]`.
   - Full GPU execution → add `[integration]`.
   - Otherwise → pick or coin a per-component tag (`[<component>]`).
5. Append the file path to `TEST_SOURCES` in `CMakeLists.txt:371-449` (inside
   the `# cmake-format: off` / `# cmake-format: on` fence). Keep alphabetical
   order.
6. Build and run the new tag:
   ```bash
   CMAKE_BUILD_PARALLEL_LEVEL=$(nproc) make
   build/release/extension/sirius/test/cpp/sirius_unittest "[your_tag]"
   ```
7. Run the regression gate before committing:
   ```bash
   build/release/extension/sirius/test/cpp/sirius_unittest "[expression_executor]"
   ```

---

*Testing analysis: 2026-05-20*
