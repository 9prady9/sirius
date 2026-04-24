# Coding Conventions

**Analysis Date:** 2026-05-20

Sirius is a DuckDB extension built in C++20 with CUDA 12/13. Two namespaces coexist
in the source tree: `sirius::` (Super Sirius — active development) and `duckdb::`
(legacy `gpu_processing` path, frozen for the v1.0 milestone). All new code lands
under `sirius::`. Legacy code under `src/legacy/` is not touched by Phase 7b
commits.

## Naming Patterns

**Files (Super Sirius — active):**
- Physical operators: `sirius_physical_<op>.{hpp,cpp}` under `src/op/` and
  `src/include/op/` (e.g., `src/op/sirius_physical_filter.cpp`,
  `src/include/op/sirius_physical_filter.hpp`).
- Plan builders: `sirius_plan_<op>.cpp` under `src/planner/` (e.g.,
  `src/planner/sirius_plan_filter.cpp`, `src/planner/sirius_plan_aggregate.cpp`).
- Plan generator entry point: `src/planner/sirius_physical_plan_generator.cpp`.
- Expression IR: `src/include/expression/ast/<node>.hpp` (one header per AST node
  kind: `between.hpp`, `case_expr.hpp`, `cast.hpp`, `coalesce.hpp`,
  `comparison.hpp`, `conjunction.hpp`, `constant.hpp`, `function_call.hpp`,
  `in_list.hpp`, `reference.hpp`, `unary_op.hpp`) plus the variant umbrella in
  `src/include/expression/ast/node.hpp`.
- Closed enums and value carriers (Phase 2 / Phase 3): `src/include/expression/value.hpp`
  and `src/include/expression/function_id.hpp`; impls in `src/expression/value.cpp`
  and `src/expression/function_id.cpp`.
- Expression executor: `src/expression_executor/gpu_expression_executor.cpp` and
  `src/expression_executor/gpu_expression_translator.cpp`; per-node specializations
  in `src/expression_executor/specializations/gpu_execute_<node>.cpp`.
- CUDA kernels: `<name>.cu` under `src/cuda/<subarea>/` (e.g.,
  `src/cuda/scan/gpu_decode_strings.cu`) and a single colocated kernel at
  `src/op/scan/equality_delete_mask.cu`.
- Test files: `test_<component>.cpp` mirroring the source layout
  (e.g., `test/cpp/expression/test_ast_scaffold.cpp`,
  `test/cpp/expression/test_function_id.cpp`,
  `test/cpp/expression/test_value.cpp`,
  `test/cpp/operator/test_physical_filter.cpp`).

**Files (legacy — frozen):**
- Operators: `gpu_physical_<op>.{hpp,cpp}` under `src/legacy/operator/` (impls)
  and `src/include/legacy/operator/` (headers).
- Plan builders: `gpu_plan_<op>.cpp` under `src/legacy/plan/`.
- CUDA kernels: `.cu` files under `src/legacy/cuda/`.
- Legacy build is gated behind `-DENABLE_LEGACY_SIRIUS=ON` (see
  `src/legacy/CMakeLists.txt`). No commits land here during v1.0.

**Functions and methods:**
- Free and member functions: lowercase `snake_case`
  (`initialize_memory_manager()`, `from_duckdb_function_name()`,
  `to_duckdb_function_name()`, `execute()`, `select()`, `holds<T>()`,
  `get<T>()`).
- Boundary translators are paired: `from_duckdb(...)` / `to_duckdb(...)` for
  payload converters (e.g., `sirius::from_duckdb`, `sirius::to_duckdb` for
  `sirius::value`).
- DuckDB-compat wrappers retain DuckDB's `PascalCase` (e.g.,
  `InitGlobalLogger`, `ParseLogLevel`, `SetGlobalLogLevel`) when they live in
  `namespace duckdb` for shim purposes.

**Variables:**
- Locals and members: lowercase `snake_case`
  (`input_batch`, `output_batches`, `column_index`, `target_type`, `try_cast`).
- Static/const globals and config knobs: `UPPER_SNAKE_CASE`
  (`Config::LOG_LEVEL`, `Config::LOG_DIR`, `Config::LOG_FLUSH_SECONDS`,
  `Config::EXPRESSION_EXECUTOR_STRATEGY`, `SIRIUS_UNITTEST_LOG_DIR`,
  `SIRIUS_PROJECT_ROOT`).
- Test-scope constexpr tables: `k_<name>` prefix
  (`k_all_comparisons`).
- Private members in C++ classes that follow the legacy DuckDB-style use the
  leading underscore (e.g., `_strategy`, `_min_ast_size`, `_temp_scalars` in
  `gpu_expression_executor`). New Super Sirius types (e.g., `sirius::ast::node`,
  `sirius::value`) keep plain member names without the underscore prefix.

**Types:**
- Classes and structs: `snake_case`
  (`sirius_physical_filter`, `gpu_expression_executor`, `shared_test_env`,
  `comparison_type`, `expression_executor_strategy`).
- Enum classes: `snake_case` type with lowercase members
  (`enum class function_id : uint16_t { add, sub, mul, ... }`,
  `enum class comparison_type : uint8_t { equal, ... }`,
  `enum class expression_executor_strategy { ... }`).
- Value carrier structs in `sirius::` namespace: `<purpose>_value` or
  `decimal<N>` suffix (`null_value`, `date_value`, `timestamp_us_value`,
  `decimal32`, `decimal64`, `decimal128`).
- Template parameters: `PascalCase` (`TestType`, `Traits`).

**Namespaces:**
- Active code: `sirius` with domain-specific nesting — `sirius::op`,
  `sirius::ast`, `sirius::memory`, `sirius::pipeline`, `sirius::test`,
  `sirius::test::operator_utils`.
- Legacy code: `duckdb` (DuckDB-extension contract).
- No mixing: a Super Sirius operator never opens `namespace duckdb` for its own
  declarations; it `#include`s DuckDB headers and references them through
  qualified names (`duckdb::Expression`, `duckdb::Connection`).

## Closed Enums and Sum Types

Phase 2 (sirius::value) and Phase 3 (sirius::function_id) introduced closed
identifier and payload types that lock the Sirius-side ABI:

**`sirius::function_id`** (`src/include/expression/function_id.hpp`):
- `enum class` backed by `uint16_t`, exactly 27 entries, grouped by category
  (arithmetic, string, temporal, struct/control). New entries go at the end of
  their category; integer values are part of the public ABI.
- Bidirectional name mapping at the DuckDB boundary:
  `from_duckdb_function_name(std::string_view) -> std::optional<function_id>`
  and `to_duckdb_function_name(function_id) -> std::string_view`.
- ABI cardinality is locked at compile time in `test/cpp/expression/test_function_id.cpp`
  via `static_assert(static_cast<uint16_t>(function_id::error) + 1 == 27, ...)`.

**`sirius::value`** (`src/include/expression/value.hpp`):
- `using value = std::variant<null_value, bool, int8_t, ..., decimal128, std::string>`
  with exactly 21 alternatives.
- `null_value` is alternative 0 so `value{}` default-constructs to typed NULL;
  the SQL type of a NULL is recovered from the sibling `sirius::logical_type`.
- HUGEINT/UHUGEINT narrow to `int64_t`/`uint64_t` at the DuckDB boundary
  (matches the executor's cuDF mapping).
- DECIMAL precision routes to `decimal32`/`decimal64`/`decimal128` by width
  class; precision lives in the companion `logical_type`.

**`sirius::ast::node`** (`src/include/expression/ast/node.hpp`):
- A struct wrapping `std::variant<reference, constant, comparison, conjunction,
  between, case_expr, cast, unary_op, coalesce, in_list, function_call>` —
  exactly 11 alternatives.
- Move-only by design; recursive children stored via `std::unique_ptr<node>`.
- Variant alternative order is part of the public ABI: new alternatives must be
  appended.
- Each per-node header forward-declares `struct node;` so it can hold
  `std::unique_ptr<node>` children before the variant is instantiated.

## AST Visitor Pattern

AST traversal uses `std::variant` + `std::visit` (no virtuals, no class
hierarchy, no enum + downcast):

```cpp
// query a node
if (n.holds<sirius::ast::comparison>()) {
  auto& c = n.get<sirius::ast::comparison>();
  // ...
}

// dispatch over all alternatives (Phase 5 dual-path executor)
std::visit([&](auto const& alt) { /* per-alternative logic */ }, n.v);
```

Compile-time invariants are verified in `test/cpp/expression/test_ast_scaffold.cpp`:
```cpp
static_assert(!std::is_copy_constructible_v<node>);
static_assert(std::is_move_constructible_v<node>);
static_assert(std::variant_size_v<node::variant_t> == 11);
```

## Code Style

**Formatter:** `clang-format` with `.clang-format` at the repo root. Key
settings:
- `BasedOnStyle: Google` with WebKit braces (`BreakBeforeBraces: WebKit`) —
  `if`/`else`/`for`/`while` braces on the same line; function and class braces
  on a new line.
- `ColumnLimit: 100`; `IndentWidth: 2`; `UseTab: Never`; `Standard: c++20`.
- `PointerAlignment: Left` (`int* ptr`, never `int *ptr`).
- `IncludeBlocks: Regroup` with explicit grouping (see Import Organization
  below).
- `AlignConsecutiveAssignments: true`, `AlignConsecutiveMacros: true`.
- `NamespaceIndentation: None`; `FixNamespaceComments: true` (closing
  `}` carries `// namespace foo`).

Apply locally with pre-commit:
```bash
pre-commit run -a              # all hooks, all files
pre-commit run clang-format    # only clang-format
```

**Linter:** `clang-tidy` with `.clang-tidy` at the repo root.
- Enabled families: `modernize-*`, selected `performance-*`, `clang-analyzer-*`.
- `WarningsAsErrors: '*'` — every clang-tidy diagnostic is a hard error.
- `HeaderFilterRegex: '.*cudf/cpp/(src|include).*'`.

**Disabled checks (and why):**
- `modernize-use-equals-default`, `modernize-concat-nested-namespaces` —
  auto-fixers are broken.
- `modernize-use-trailing-return-type`, `modernize-return-braced-init-list`,
  `modernize-use-bool-literals` — stylistic preference.
- `modernize-use-constraints`, `modernize-use-ranges`,
  `modernize-use-designated-initializers` — C++20-feature checks not yet
  wanted project-wide.
- `clang-analyzer-cplusplus.NewDeleteLeaks`,
  `clang-analyzer-optin.core.EnumCastOutOfRange`,
  `clang-analyzer-optin.cplusplus.UninitializedObject` — upstream LLVM bugs or
  conflicts with flag-style enum usage.

**Python:** `black` (pinned to `25.1.0` in `.pre-commit-config.yaml`). Applies
to Python tooling under `test/tpch_performance/` and elsewhere.

**CMake:** `cmake-format` and `cmake-lint` (`cheshirekow/cmake-format-precommit`
v0.6.13; lint args `--line-width 220 --disabled-codes=C0307
--suppress-decorations`). Test-source lists in `CMakeLists.txt` are wrapped in
`# cmake-format: off` / `# cmake-format: on` fences.

**Spell check:** `codespell` (`v2.4.1`) with project allowlist in
`.codespell_words` (`aktion`, `ans`, `foto`, `fpr`, `ist`, `thirdparty`).

**Other pre-commit hooks** (from `.pre-commit-config.yaml`):
- `pre-commit/pre-commit-hooks v6.0.0`: `check-added-large-files`,
  `check-case-conflict`, `check-json`, `check-merge-conflict`,
  `check-symlinks`, `check-toml`, `check-yaml`, `debug-statements`,
  `destroyed-symlinks`, `detect-private-key`, `end-of-file-fixer`,
  `mixed-line-ending`, `pretty-format-json --autofix`, `trailing-whitespace`.
- `sirosen/texthooks v0.7.1`: `fix-smartquotes`.

**Global exclusions:** `\.csv$|^legacy/queries/.*` — CSV files (often hold test
answers) and the legacy queries tree are excluded from all hooks.

## Import Organization

`clang-format` enforces include ordering via `IncludeBlocks: Regroup`. Groups,
in priority order (from `.clang-format`):

1. **Sirius local includes** (quoted) — highest priority.
   ```cpp
   #include "expression/ast/node.hpp"
   #include "expression/function_id.hpp"
   #include "op/sirius_physical_operator.hpp"
   ```
2. **Benchmark / test angle-bracket includes** (`<benchmarks/...>`, `<tests/...>`).
3. **cuDF test includes** (`<cudf_test/...>`).
4. **cuDF library includes** (`<cudf/...>`).
5. **Other libcudf** (`<nvtext/...>`, `<cudf_kafka/...>`).
6. **RAPIDS sibling libraries** (`<cugraph/...>`, `<cuml/...>`, `<raft/...>`,
   `<kvikio/...>`).
7. **RMM** (`<rmm/...>`).
8. **CCCL / CUDA core**
   (`<thrust/...>`, `<cub/...>`, `<cuda/...>`, `<cooperative_groups/...>`,
   `<cuco/...>`, `<cuda.h>`, `<cuda_runtime*>`, `<nvtx3/...>`).
9. **Other system headers with a dot** (`<sys/types.h>`, `<duckdb/.../...hpp>`).
10. **C++ standard library** (`<vector>`, `<memory>`, `<variant>`).

**Sirius headers are quoted** because the `sirius_unittest` target adds
`src/include` to `target_include_directories` (see `CMakeLists.txt:467-468`).
Inside `sirius/` and `op/` and the like, headers refer to siblings as
`#include "op/foo.hpp"`, never with angle brackets.

**DuckDB headers are angle-bracketed** as system includes
(`#include <duckdb/common/types/data_chunk.hpp>`).

## Header Layout and Module Design

- `src/include/<component>/` mirrors `src/<component>/`. Each `.cpp` has a
  matching `.hpp` in the parallel `include/` tree.
- Header guards: `#pragma once` only; no manual `#ifndef` guards.
- Per-AST-node headers (`src/include/expression/ast/*.hpp`) forward-declare
  `struct node;` rather than including `node.hpp`. The umbrella `node.hpp`
  pulls in every per-node header before defining the variant.
- Public surface for the expression IR lives in
  `src/include/expression/expression.hpp` (the Phase 7a PIMPL wrapper around
  `duckdb::Expression`). Implementation details that need DuckDB types live in
  `src/include/expression/expression_internal.hpp`.
- Closed enums and value variants live in their own headers
  (`function_id.hpp`, `value.hpp`) — kept distinct so callers can include only
  what they need.
- Barrel headers are not used: each component exposes its main type(s) via a
  dedicated header.

## Error Handling

**Sirius-native exception hierarchy** (`src/include/sirius/exception.hpp`):
```cpp
namespace sirius {
class internal_exception       : public std::runtime_error { ... };
class not_implemented_exception: public std::runtime_error { ... };
class invalid_input_exception  : public std::runtime_error { ... };
}
```
Each ctor accepts a `std::string` or a `std::format_string<Args...>` for
formatted messages. New Super Sirius code throws these. Use cases:
- `sirius::not_implemented_exception` — payload converters that hit an
  unsupported `logical_type` / `function_id` (e.g., `sirius::from_duckdb`
  on an unsupported DECIMAL width — see `src/expression/value.cpp`).
- `sirius::internal_exception` — invariant violations in Super Sirius internals.
- `sirius::invalid_input_exception` — caller-side argument validation.

**DuckDB-side exceptions:** at the plan-builder boundary and inside operators
that still consume `duckdb::Expression` (pre-Phase-7b paths), continue to throw
`duckdb::Exception` variants (e.g., `duckdb::NotImplementedException`,
`duckdb::InvalidInputException`). Don't introduce new `duckdb::*` exceptions in
code under `src/op/` or `src/expression_executor/` going forward; prefer the
`sirius::` hierarchy.

**cuDF / RMM errors** propagate as `cudf::logic_error` / `rmm::logic_error` /
`rmm::out_of_memory`. These bubble up to the operator boundary; do not swallow
them. The fallback layer (`src/sirius_interface.cpp`) catches them at the engine
boundary to trigger CPU fallback.

**Assertions:** `D_ASSERT` (DuckDB macro) for preconditions in operator and
plan-builder code (e.g., `D_ASSERT(static_cast<bool>(expression));` in
`sirius_physical_filter`). Tests use Catch2 `REQUIRE` / `CHECK`.

## Logging

**Framework:** spdlog via a thin Sirius wrapper at
`src/include/log/logging.hpp`. The wrapper hides spdlog from CUDA TUs because
nvcc cannot compile spdlog/fmt's chrono headers.

**Macros:**
```cpp
SIRIUS_LOG_TRACE(fmt, args...)
SIRIUS_LOG_DEBUG(fmt, args...)
SIRIUS_LOG_INFO(fmt, args...)
SIRIUS_LOG_WARN(fmt, args...)
SIRIUS_LOG_ERROR(fmt, args...)
SIRIUS_LOG_FATAL(fmt, args...)
```

**CUDA context:** macros expand to no-ops under `__CUDACC__`:
```cpp
#ifdef __CUDACC__
#define SIRIUS_LOG_TRACE(...)
// ...
#endif
```

**Runtime configuration:** the extension reads two env vars on context creation
(`src/sirius_context.cpp`):
- `SIRIUS_LOG_DIR` — log directory (default: `Config::LOG_DIR`, falls back to
  `${CMAKE_BINARY_DIR}/log`).
- `SIRIUS_LOG_LEVEL` — one of `trace | debug | info | warn | error |
  critical | off` (default: `info`).
- Flush interval: `Config::LOG_FLUSH_SECONDS` (default: 3 seconds).
- Sink: `spdlog::sinks::daily_file_sink_mt`, pattern
  `[%Y-%m-%d %T.%e] [%l] [%s:%#] %v` (timestamp, level, source:line, message).

**When to log:**
- `INFO` — major execution milestones (plan generation, pipeline start/end).
- `DEBUG` — operator-level execution detail, memory reservations.
- `WARN` — fallback decisions, resource pressure.
- `ERROR` — exceptions that trigger fallback.
- `TRACE` — verbose per-task or per-kernel diagnostics; CI sets
  `SIRIUS_LOG_LEVEL=trace` (see `.github/workflows/test.yml`).

## Comments

- Explain why, not what. Code shows what; comments capture intent, trade-offs,
  or invariants the type system cannot encode.
- Mark deferred work as `TODO(owner):` or `FIXME(issue-xxx):` with an owner or
  issue link.
- `@brief` / `@param` / `@return` doxygen style for public types and methods
  (see `gpu_expression_executor.hpp`, `function_id.hpp`, `value.hpp`).
  Reflow is on (`ReflowComments: true`) — don't manually wrap doxygen blocks
  past the column limit; let clang-format handle them.

## License Headers

Every C++/CUDA/Python source and header carries the Apache 2.0 block as the
first 15 lines:

```cpp
/*
 * Copyright 2025, Sirius Contributors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
```

`#pragma once` and the first `#include` come immediately after the license
block.

## Function Design

- Aim for functions under ~60 lines; split larger logic into helpers.
- Pass tables, batches, and large containers by const reference or shared
  ownership (`std::shared_ptr<cucascade::data_batch>` for data batches that
  cross operator boundaries).
- Pass GPU memory ranges via `cudf::device_span<>`, `cudf::column_view`,
  `cudf::table_view`, or `rmm::device_uvector<>`.
- DuckDB-facing types stay in `duckdb::vector<T>` (not `std::vector<T>`) at
  operator constructor signatures — preserves DuckDB's allocator integration.
  Inside the operator body, plain `std::vector<T>` is fine.
- Streams: every cuDF/RMM-touching API takes `rmm::cuda_stream_view` as an
  explicit parameter (default `cudf::get_default_stream()`).
- Memory resources: default `rmm::device_async_resource_ref` is
  `cudf::get_current_device_resource_ref()`.

## CUDA File Layout

- Active `.cu` files (compiled into the main `sirius_extension` library):
  - `src/cuda/scan/gpu_decode_alp.cu`
  - `src/cuda/scan/gpu_decode_bitpacking.cu`
  - `src/cuda/scan/gpu_decode_rle.cu`
  - `src/cuda/scan/gpu_decode_strings.cu`
  - `src/cuda/scan/gpu_native_decode.cu`
  - `src/op/scan/equality_delete_mask.cu` (colocated with its plan-builder
    consumer)
- Legacy `.cu` files live under `src/legacy/cuda/` and are only compiled when
  `ENABLE_LEGACY_SIRIUS=ON`.
- CUDA standard: 20 (matches C++20). Separable compilation is enabled
  (`CMAKE_CUDA_SEPARABLE_COMPILATION ON`). Target GPU arches:
  `75;80;86;90a;100f;120a;120` (Turing through Blackwell).
- `.cu` translation units must use the no-op `SIRIUS_LOG_*` macros; CUDA TUs
  cannot include spdlog.

## Move Semantics

Types that own GPU resources, file handles, or recursive children are
move-only. The contract is verified at compile time in tests:

```cpp
static_assert(!std::is_copy_constructible_v<sirius::ast::node>);
static_assert(std::is_move_constructible_v<sirius::ast::node>);
```

Move-only types in active code:
- `sirius::ast::node` (variant of unique_ptr-holding structs)
- `sirius::expression` (PIMPL wrapper)
- `sirius::test::shared_test_env` (owns the DuckDB instance)

## Namespace Policy

- **`sirius`** — Super Sirius, all active development. Every new file lands here.
- **`sirius::<component>`** — domain-specific
  (`sirius::ast`, `sirius::op`, `sirius::memory`, `sirius::pipeline`,
  `sirius::test`, `sirius::test::operator_utils`).
- **`duckdb`** — only legacy code under `src/legacy/` and the DuckDB-extension
  registration shim (`SiriusContextExtensionCallback` in
  `src/sirius_context.cpp`). The legacy namespace is frozen for the v1.0
  milestone; no Phase 7b commit changes anything under `src/legacy/` or any
  `namespace duckdb { ... }` block in active sources.

---

*Convention analysis: 2026-05-20*
