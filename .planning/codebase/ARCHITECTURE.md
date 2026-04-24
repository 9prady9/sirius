# Architecture

**Analysis Date:** 2026-05-20

> **Canonical reference:** Architectural narrative lives in `docs/super-sirius/` — start at `docs/super-sirius/README.md`. This document is the codebase-map summary; the in-tree docs go deeper. Of particular relevance:
> - `docs/super-sirius/architecture-overview.md` — component diagram, thread model, ownership
> - `docs/super-sirius/execution-flow.md` — end-to-end query trace
> - `docs/super-sirius/physical-plan-generation.md` — DuckDB→Sirius plan translation
> - `docs/super-sirius/expression-executor.md` — GPU expression evaluation
> - `docs/super-sirius/pipeline-execution.md`, `task-creator.md`, `memory-management.md`, `data-management.md`, `scan.md`

## Pattern Overview

**Overall:** Sirius is a **DuckDB extension** that intercepts DuckDB's physical plan and routes supported operators to **Super Sirius** — a multi-pipeline, task-based GPU execution engine backed by cuDF / RMM / cuCascade. The engine itself is a directed graph of split pipelines whose tasks are dispatched to dedicated GPU/CPU thread pools, with tiered (GPU/Host/Disk) memory managed by cuCascade.

There are **two execution paths** in the codebase, and they do not share code:

| Aspect            | **Super Sirius** (active, all new work)               | **Legacy** (frozen, kept for compat)                  |
|-------------------|-------------------------------------------------------|-------------------------------------------------------|
| Namespace         | `sirius`                                              | `duckdb` (`duckdb::gpu*`)                             |
| Entry point       | Plain SQL (transparent) or `CALL gpu_execution(...)`  | `CALL gpu_buffer_init(...)` then `CALL gpu_processing(...)` |
| Source root       | `src/` (top-level), `src/include/` (top-level)        | `src/legacy/`, `src/include/legacy/`                  |
| Build flag        | Always built                                          | `-DENABLE_LEGACY_SIRIUS=ON` (off by default)          |
| Plan generator    | `sirius::planner::sirius_physical_plan_generator`     | `duckdb::GPUPhysicalPlanGenerator`                    |
| Operator base     | `sirius::op::sirius_physical_operator`                | `duckdb::GPUPhysicalOperator`                         |
| Execution model   | Multi-pipeline, task-based, multi-threaded            | Single-threaded GPU executor                          |
| Memory manager    | cuCascade tiered (GPU/Host/Disk) + RMM                | `GPUBufferManager` (pinned host buffers)              |
| Expression engine | `sirius::gpu_expression_executor` (AST_INTERPRET / AST_JIT / MATERIALIZE) | `duckdb::gpu_expression_executor` (legacy specializations) |
| Fallback          | `transparent::PhysicalSiriusExecution` plus `sirius_interface` exception path | `src/legacy/fallback.cpp` (`FallbackChecker`) |

**Key Characteristics (Super Sirius):**
- **DuckDB optimizer hook** captures the logical plan; **`OnFinalizePrepare`** on `SiriusContext` swaps DuckDB's physical plan for a `transparent::PhysicalSiriusExecution` wrapping the Sirius plan. The explicit `CALL gpu_execution('...')` table function is the same path with no plan swap.
- **Logical-to-physical translation** is the only place that touches `duckdb::Bound*Expression`. Below the plan-builder boundary, the codebase progressively dispatches on Sirius-native types: `sirius::expression` (Phase 7a PIMPL wrapper, currently still over a `duckdb::Expression`), `sirius::ast::node` variant (Phases 1-3 landed; Phase 4 wires up `sirius::ast::from_duckdb`), `sirius::value` (constant payload), `sirius::function_id` (function dispatch enum).
- **Operator-driven pipeline splitting**: scanning, hash joins, grouping, sorting, top-N split into multiple split pipelines via `sirius_meta_pipeline` → `sirius_pipeline_converter`.
- **Task-based execution**: scan tasks land in `duckdb_scan_executor`; GPU pipeline tasks land in `gpu_pipeline_executor`; downgrade tasks land in `downgrade_executor`. The `task_creator` follows hint chains to schedule the next-ready operator.
- **Multi-threaded**: DuckDB query thread, task-scheduler management thread, per-GPU manager + worker thread pool, scan manager + workers, task-creator pool, downgrade monitor + workers.
- **Tiered memory** via cuCascade: GPU memory space + Host memory space + (optional) disk; RMM owns GPU allocations; `sirius_memory_reservation_manager` brokers reservations.
- **Graceful CPU fallback** when an operator/type isn't supported, the libcudf int32 row-count limit (~2B) is exceeded, or GPU memory is too small even after spilling.

## Layers

**Extension & Binding:**
- Purpose: DuckDB extension loading, optimizer hook registration, `gpu_execution` table function, legacy `gpu_processing` shims.
- Location: `src/sirius_extension.cpp`, `src/include/sirius_extension.hpp`, `src/transparent/sirius_optimizer_extension.cpp`, `src/transparent/physical_sirius_execution.cpp`, `src/include/transparent/`.
- Contains: `SiriusExtension::Load`, `GPUExecutionFunction`, transparent optimizer extension that copies the logical plan into `SiriusContext`, `PhysicalSiriusExecution` (`sirius::transparent::PhysicalSiriusExecution`) that replaces DuckDB's CPU plan.
- Depends on: DuckDB extension/optimizer/binder APIs, planner layer, interface layer.
- Used by: DuckDB's prepare/execute path.

**Query Interface:**
- Purpose: Per-connection query lifecycle — parse / prepare / execute / fetch with fallback signalling.
- Location: `src/sirius_interface.cpp`, `src/include/sirius_interface.hpp`.
- Contains: `sirius::sirius_interface` (pending-statement state machine), error finalization, fetch loop, telemetry context creation.
- Depends on: DuckDB client context, `sirius_engine`, telemetry.
- Used by: `GPUExecutionFunction`, `PhysicalSiriusExecution::GetData`.

**Planning Layer:**
- Purpose: Translate DuckDB optimized logical plans to Sirius physical operator trees; the **only** place that consumes `duckdb::Bound*Expression`.
- Location: `src/planner/`, `src/include/planner/`.
- Contains: `sirius_physical_plan_generator::create_plan()` (main entry), per-construct plan builders `sirius_plan_aggregate.cpp`, `sirius_plan_comparison_join.cpp`, `sirius_plan_delim_join.cpp`, `sirius_plan_filter.cpp`, `sirius_plan_projection.cpp`, `sirius_plan_order.cpp`, `sirius_plan_top_n.cpp`, `sirius_plan_limit.cpp`, `sirius_plan_cte.cpp`, `sirius_plan_recursive_cte.cpp`, `sirius_plan_get.cpp`, `sirius_plan_column_data_get.cpp`, `sirius_plan_delim_get.cpp`, `sirius_plan_dummy_scan.cpp`, `sirius_plan_empty_result.cpp`, `sirius_plan_expression_get.cpp`. `src/planner/query.cpp` holds the per-query planning context.
- Depends on: DuckDB logical operator tree, `sirius::expression` PIMPL wrapper (today), `sirius::ast::from_duckdb` (Phase 4, in flight), the scan-manager layer for `Get` operators.
- Used by: `OnFinalizePrepare` (transparent path), `sirius_engine::initialize` (explicit `gpu_execution`).

**Engine / Pipeline Construction:**
- Purpose: Orchestrate pipeline construction from a finished physical plan; own the per-query telemetry handle.
- Location: `src/sirius_engine.cpp`, `src/include/sirius_engine.hpp`, `src/pipeline/sirius_meta_pipeline.cpp`, `src/pipeline/sirius_pipeline_converter.cpp`, `src/pipeline/sirius_pipeline.cpp`, `src/pipeline/repository_wiring_materializer.cpp`, `src/pipeline/sirius_plan_printer.cpp`.
- Contains: `sirius::sirius_engine` (build / execute / cancel), `sirius_meta_pipeline` (recursive logical builder), `sirius_pipeline_converter` (splits meta pipelines into a multi-pipeline graph and wires `repository_wiring`), `sirius_pipeline` (executable unit: source + middle ops + sink + ports), plan printer for `EXPLAIN`-style diagnostics.
- Depends on: planning layer, scan manager, task scheduler, `SiriusContext` subsystems.
- Used by: `sirius_interface::sirius_execute_query`.

**Pipeline Execution:**
- Purpose: Dispatch GPU pipeline tasks, manage CUDA streams, retry on OOM.
- Location: `src/pipeline/task_scheduler.cpp`, `src/pipeline/gpu_pipeline_executor.cpp`, `src/pipeline/gpu_pipeline_task.cpp`, `src/pipeline/task_request.cpp`, `src/include/pipeline/`.
- Contains: `sirius::task_scheduler` (top-level orchestrator + management loop + `completion_handler`), `gpu_pipeline_executor` (per-GPU manager + worker pool, kiosk ticketing for reservations), `gpu_pipeline_task` (carries pipeline + input batches + stream), `oom_reschedule_exception`, `pipeline_memory_history`, `pipeline_build_context`.
- Depends on: data repositories, memory reservation manager, task creator.
- Used by: `sirius_engine::execute`.

**Task Creation & Scheduling:**
- Purpose: Decide which operator is next-ready after each task completes.
- Location: `src/creator/task_creator.cpp`, `src/include/creator/task_creator.hpp`.
- Contains: hint-chain following, per-operator readiness rules, recursive producer lookup, task-creation queue plumbing.
- Depends on: operators' `get_next_task_hint()`, data ports, repository manager.
- Used by: scan executor + gpu pipeline executor on task completion.

**Scan / Data Ingestion:**
- Purpose: Read data from DuckDB tables, Parquet files (local + S3 + io_uring backed), Iceberg tables, into GPU-compatible data batches.
- Location: `src/op/scan/`, `src/include/op/scan/`, `src/cuda/scan/`, `src/scan_manager/`, `src/include/scan_manager/`, `src/io/`, `src/include/io/`, `src/op/sirius_physical_table_scan.cpp`, `src/op/sirius_physical_parquet_scan.cpp`, `src/op/sirius_physical_iceberg_scan.cpp`, `src/op/sirius_physical_duckdb_scan.cpp`, `src/op/sirius_physical_column_data_scan.cpp`, `src/op/sirius_physical_cpu_source.cpp`.
- Contains: `duckdb_scan_executor` (the scan-executor thread pool), `duckdb_scan_task` / `parquet_scan_task` / `iceberg_scan_task` / `cpu_source_task`, scan caching (`cached_ranges`, `cached_split_provider`), Parquet metadata + schema mapping, native DuckDB decode walker, GPU decoders in `src/cuda/scan/` (ALP, bitpacking, RLE, strings, generic native decode), Iceberg metadata reader, equality + positional delete filters, puffin reader, scan manager (`sirius_scan_manager`) with split connectors, io subsystem (`io_context`, S3 client + sigv4, io_uring reactor + ioctx, prefetching cache, URI parser, datasource factory, admission control).
- Depends on: DuckDB table functions, Parquet readers, cuDF column factories, RMM, cuCascade host/pinned memory.
- Used by: initial pipelines after `task_scheduler::start_query`.

**Operators (GPU-native):**
- Purpose: Run SQL operators on GPU via cuDF + custom CUDA kernels.
- Location: `src/op/` plus `aggregate/`, `merge/`, `order/`, `partition/`, `result/`, `scan/` subdirectories; mirrored headers in `src/include/op/`.
- Contains: 30+ operator implementations including `sirius_physical_hash_join`, `sirius_physical_nested_loop_join`, `sirius_physical_delim_join`, `sirius_physical_grouped_aggregate` + `_merge`, `sirius_physical_ungrouped_aggregate` + `_merge`, `sirius_physical_order`, `sirius_physical_merge_sort`, `sirius_physical_sort_partition`, `sirius_physical_sort_sample`, `sirius_physical_top_n` + `_merge`, `sirius_physical_filter`, `sirius_physical_projection`, `sirius_physical_limit`, `sirius_physical_concat`, `sirius_physical_partition` + `_consumer_operator`, `sirius_physical_result_collector`, `sirius_physical_cte`, `sirius_physical_dummy_scan`, `sirius_physical_empty_result`, `sirius_dynamic_filter` (runtime filter shared across operators). GPU-side helpers in `op/aggregate/gpu_aggregate_impl.cpp`, `op/order/gpu_order_impl.cpp`, `op/merge/gpu_merge_impl.cpp`, `op/partition/gpu_partition_impl.cpp`, and `op/result/host_table_chunk_reader.cpp`.
- Depends on: cuDF, `gpu_expression_executor`, data repositories, memory reservation manager.
- Used by: `gpu_pipeline_executor` (one operator per call inside a task).

**Expression Execution (GPU):**
- Purpose: Evaluate SQL expressions (filter predicates, projections, join conditions, aggregate inputs) on GPU.
- Location: `src/expression_executor/`, `src/include/expression_executor/`, plus the new Sirius-native AST under `src/expression/` and `src/include/expression/`.
- Contains:
  - `gpu_expression_executor` — main dispatcher with three strategies: `MATERIALIZE`, `AST_INTERPRET`, `AST_JIT` (see `expression_executor_strategy.cpp` and `expression_executor_strategy.hpp`).
  - `gpu_expression_translator` — translates a Sirius expression tree into a `cudf::ast::tree` for AST-mode evaluation.
  - Per-node specializations in `src/expression_executor/specializations/`: `gpu_execute_between.cpp`, `gpu_execute_case.cpp`, `gpu_execute_cast.cpp`, `gpu_execute_comparison.cpp`, `gpu_execute_conjunction.cpp`, `gpu_execute_constant.cpp`, `gpu_execute_function.cpp` (now dispatches on `sirius::function_id`), `gpu_execute_operator.cpp`, `gpu_execute_reference.cpp`.
  - Regex playground in `src/expression_executor/regex/regex_playground.cpp`.
  - AST allowlist in `src/include/expression_executor/ast_supported_types.hpp` (per `sirius::type_id`, replacing the old `duckdb::LogicalTypeId`-keyed list).
  - Phase 7a wrapper `sirius::expression` (PIMPL over `duckdb::Expression`) in `src/expression/expression.cpp`, `src/include/expression/expression.hpp`, `src/include/expression/expression_internal.hpp`, plus the mirror struct `sirius::join_condition` in `src/expression/join_condition.cpp`.
  - Phase 1-3 Sirius-native scaffolding:
    - `sirius::ast::node` variant in `src/include/expression/ast/node.hpp`, with alternatives `reference`, `constant`, `comparison`, `conjunction`, `between`, `case_expr`, `cast`, `unary_op`, `coalesce`, `in_list`, `function_call` (one header per kind).
    - `sirius::value` variant for constant payloads in `src/include/expression/value.hpp` / `src/expression/value.cpp` (carriers for `null_value`, `date_value`, `timestamp_*_value`, plus the primitive numeric and string variants).
    - `sirius::function_id` closed enum (27 entries: arithmetic 6, string 9, temporal 9, struct/control 3) in `src/include/expression/function_id.hpp` / `src/expression/function_id.cpp`, with `from_duckdb_function_name` / `to_duckdb_function_name` round-trip helpers.
- Depends on: cuDF binary/unary/regex/datetime ops, RMM, the Sirius type system (`helper/logical_type.hpp`, `helper/types.hpp`).
- Used by: filter, projection, hash join (build/probe key + condition), nested loop join, grouped aggregate (group key + aggregate inputs), order/top-N (sort keys).

**Memory Management:**
- Purpose: GPU/Host/Disk memory reservations and pressure-driven spilling.
- Location: `src/memory/sirius_memory_reservation_manager.cpp`, `src/memory/defragmenter_oom_policy.cpp`, `src/downgrade/downgrade_executor.cpp`, headers in `src/include/memory/` and `src/include/downgrade/`.
- Contains: `sirius::sirius_memory_reservation_manager` (tier reservations, per-task tickets), `defragmenter_oom_policy`, `multiple_blocks_allocation_accessor`, `host_table_utils`, `resource_ref_utils`, `downgrade_executor` (monitor loop + worker pool that moves data GPU→Host).
- Depends on: cuCascade memory spaces, RMM, data repository manager.
- Used by: `gpu_pipeline_executor` (reservation acquisition), scan executor (host buffer reservation), every operator that allocates.

**Data Management:**
- Purpose: Central registry for data batches and convertible-data plumbing.
- Location: `src/data/host_parquet_representation.cpp`, `src/data/host_parquet_representation_converters.cpp`, headers in `src/include/data/`.
- Contains: `convertible_data` / `convertible_data_batch` / `convertible_gpu_pipeline_task`, `cached_data_representation`, `host_parquet_representation`, `data_batch_utils`, `sirius_converter_registry` (type conversion registry — registered in `src/sirius_interface.cpp`).
- Depends on: cuCascade data repository manager + memory spaces, RMM, cuDF.
- Used by: all operators (through repositories), scan executor, downgrade executor, expression executor.

**Context & Configuration:**
- Purpose: Per-DuckDB-connection lifetime; runtime tuning.
- Location: `src/sirius_context.cpp`, `src/include/sirius_context.hpp`, `src/sirius_config.cpp`, `src/include/sirius_config.hpp`, `src/config.cpp`, `src/include/config.hpp`, telemetry in `src/telemetry/telemetry_context.cpp` + `src/include/telemetry/telemetry_context.hpp`.
- Contains: `duckdb::SiriusContext` (`ClientContextState` subclass; owns memory manager, repository manager, task scheduler, downgrade executors, task creator, scan manager, telemetry; implements `OnFinalizePrepare` for transparent execution), `sirius::sirius_config` (thread pool sizes, memory limits, operator parameters), config loader, YAML-backed `~/.sirius/sirius.yaml` + legacy `sirius.cfg`, `telemetry_context` (quent-backed query telemetry).
- Depends on: every other subsystem.
- Used by: extension load, every query.

**Exec / Concurrency Primitives:**
- Purpose: Shared async building blocks consumed by scan, downgrade, and task scheduler.
- Location: `src/include/exec/` (header-only).
- Contains: `bounded_thread_pool`, `thread_pool`, `channel`, `inspectable_mpsc`, `interruptible_mpmc`, `scoped_dispatcher`, generic `exec/config.hpp`.

**Legacy (Isolated):**
- Purpose: Old single-threaded `gpu_processing` path; kept for compatibility but no new development.
- Location: `src/legacy/` (sources) and `src/include/legacy/` (headers), with `src/legacy/CMakeLists.txt` gating compilation on `-DENABLE_LEGACY_SIRIUS=ON`.
- Contains: `duckdb::GPUPhysicalPlanGenerator`, `GPUPhysicalOperator`, `gpu_executor`, `gpu_buffer_manager`, `gpu_meta_pipeline`, `gpu_pipeline`, `gpu_query_result`, `gpu_columns`, `gpu_context`, plus legacy operators (`src/legacy/operator/gpu_physical_*.cpp` — hash/nested-loop join, grouped/ungrouped aggregate, order/top-n, filter, projection, partition, CTE, delim join, strings matching, substring, table scan, column-data scan, dummy scan, empty result, result collector, materialize), legacy plan builders (`src/legacy/plan/gpu_plan_*.cpp`), legacy expression executor (`src/legacy/expression_executor/gpu_expression_executor.cpp` + `_state.cpp` + `specializations/gpu_execute_*.cpp`), legacy CUDA kernels (`src/legacy/cuda/operator/*.cu`, `src/legacy/cuda/cudf/*.cu`, `src/legacy/cuda/expression_executor/*.cu`, `src/legacy/cuda/allocator.cu`, `communication.cu`, `print.cu`, `utils.cu`), `src/legacy/cpu_cache.cpp` (host-side cache), and `src/legacy/fallback.cpp` (`duckdb::FallbackChecker`).
- Build control: `-DENABLE_LEGACY_SIRIUS=ON` (off by default). When off, none of `src/legacy/` is compiled.

## Data Flow

**Query Ingestion & Planning:**

1. SQL string arrives at a DuckDB client.
2. DuckDB optimizer calls `sirius_pre_optimizer_hook` (registered by `sirius::transparent::sirius_optimizer_extension`) which snapshots which optimizers DuckDB has disabled.
3. DuckDB optimizer produces the optimized logical plan.
4. DuckDB optimizer calls `sirius_optimizer_hook`, which deep-copies the logical plan into `duckdb::SiriusContext`.
5. DuckDB's physical planner generates a CPU physical plan.
6. DuckDB calls `SiriusContext::OnFinalizePrepare`:
   - Retrieves the saved logical plan.
   - Calls `sirius::planner::sirius_physical_plan_generator::create_plan()` to convert it to a Sirius physical operator tree (this is where the only `duckdb::Bound*Expression` → `sirius::expression` / `sirius::ast::node` translation happens).
   - On success, constructs `sirius::transparent::PhysicalSiriusExecution` and instructs DuckDB to rebind the prepared statement, replacing the CPU plan.
   - On failure (unsupported op/type, fallback signal), the CPU plan is left in place — DuckDB executes the query on CPU.

   Alternative entry: `CALL gpu_execution('SELECT ...')` (`SiriusExtension::GPUExecutionFunction` in `src/sirius_extension.cpp`) re-parses + re-optimizes inside the table function and runs the same planner.

**Execution Orchestration:**

7. DuckDB executor calls `PhysicalSiriusExecution::GetDataInternal` (transparent) or `GPUExecutionFunction` (explicit). Both construct a `sirius::sirius_interface` and call `sirius_execute_query()`.
8. `sirius_interface` creates a `sirius::sirius_engine`, hands it the Sirius physical plan.
9. `sirius_engine::initialize_internal()` builds meta-pipelines (`sirius_meta_pipeline::build()` + `ready()`), splits them via `sirius_pipeline_converter` (operators that split: TABLE_SCAN family, HASH_JOIN, GROUPED_AGGREGATE, ORDER_BY, MERGE_SORT, SORT_PARTITION/SAMPLE, TOP_N, PARTITION), injects PARTITION / CONCAT / MERGE operators at boundaries, and wires data repositories with `MemoryBarrierType` (FULL / PARTIAL / PIPELINE) via `repository_wiring_materializer`.
10. `sirius_engine::execute()` calls `task_scheduler::prepare_for_query()` then `start_query()`, which distributes a `completion_handler` to every sub-executor and queues the initial scan operators.

**Scan Phase:**

11. The scan executor pops scan tasks, acquires pinned host memory + GPU streams, reads from DuckDB tables / Parquet files / Iceberg tables (with optional S3 + io_uring + prefetching cache + caching by `cached_ranges`).
12. Decoded data is converted to GPU column format and published as `data_batch`es into the consumer's repository.
13. The scan task calls `task_creator->schedule(downstream_op)` to mark the consumer as ready.

**Pipeline Execution Loop:**

14. The task creator pops the hint chain to find a READY operator.
15. It enqueues a GPU pipeline task (source operator → middle operators → sink operator) onto a `gpu_pipeline_executor`.
16. The GPU executor acquires a CUDA stream + memory reservation and runs the pipeline:
    - Locks input batches, migrates them to GPU if necessary.
    - Calls `execute()` on each operator in order (`sirius_physical_operator::execute` and overrides).
    - Calls the sink's `sink()` to publish results into downstream ports.
    - On `oom_reschedule_exception`, retries up to 10 times with 5ms back-off; on other exceptions, reports to `completion_handler->report_error()`.
17. On task completion, the executor calls `task_creator->schedule(next_downstream_op)`.
18. The loop continues until the result-collector pipeline finishes and `completion_handler::mark_completed()` signals done.

**Memory Management Loop (concurrent):**

19. Each `downgrade_executor` (one per memory space) polls cuCascade pressure every ~10ms.
20. When the high-water threshold trips, the monitor enqueues downgrade tasks; the worker pool moves data GPU→Host (and Host→Disk if configured), updating reservations.

**Result Extraction:**

21. The DuckDB query thread unblocks on the future.
22. `sirius_interface::fetch_result_internal()` extracts the `MaterializedQueryResult` from the result collector operator.
23. DuckDB streams the result back to the client.

**Error / Fallback Handling:**

24. Any executor exception is caught and propagated to `completion_handler->report_error()`. The pipeline executor drains the task queue, stops the task creator, and waits for all sub-executors to drain.
25. If `fallback_enabled` is set, `sirius_interface` catches the exception and re-runs the query through DuckDB's CPU executor; otherwise the error is returned to the caller.
26. Plan-time fallback: if the planner cannot translate an operator/type/expression, the Sirius plan is rejected and DuckDB's CPU plan stays in effect (transparent path) or `CALL gpu_execution(...)` returns a not-implemented error.

## Key Abstractions

**`sirius::op::sirius_physical_operator`:**
- Purpose: Base class for every Super Sirius GPU operator.
- Examples: `src/op/sirius_physical_hash_join.cpp`, `src/op/sirius_physical_grouped_aggregate.cpp`, `src/op/sirius_physical_order.cpp`, `src/include/op/sirius_physical_operator.hpp`.
- Pattern: Subclasses own DuckDB-derived metadata (column bindings, expressions) plus operator-specific state; `execute(...)` runs per-pipeline GPU work on a cuDF/RMM stream; `sink()` flushes into downstream ports; `get_next_task_hint()` drives task creation.

**`sirius::sirius_meta_pipeline`:**
- Purpose: Recursive logical pipeline grouping before multi-pipeline splitting.
- Examples: `src/pipeline/sirius_meta_pipeline.cpp`, `src/include/pipeline/sirius_meta_pipeline.hpp`.
- Pattern: `build(op)` walks the physical tree; `ready()` reverses operator lists; `get_pipelines()` produces ordered pipelines for the converter.

**`sirius::sirius_pipeline`:**
- Purpose: Executable pipeline (source + middle operators + sink + ports).
- Examples: `src/pipeline/sirius_pipeline.cpp`, `src/include/pipeline/sirius_pipeline.hpp`.
- Pattern: Holds operator list, sink reference, source reference, port map, repository wiring, and barrier semantics.

**`sirius::gpu_pipeline_task`:**
- Purpose: A unit of GPU work scheduled on a CUDA stream.
- Examples: `src/pipeline/gpu_pipeline_task.cpp`, `src/include/pipeline/gpu_pipeline_task.hpp`, `src/include/pipeline/sirius_pipeline_itask.hpp`, `src/include/pipeline/sirius_pipeline_task_states.hpp`.
- Pattern: Wraps a pipeline + input data-batches; consumes a memory reservation; catches `oom_reschedule_exception` for retry; reports completion to the task creator.

**`sirius::data_batch` (cuCascade-backed):**
- Purpose: Column-oriented data with GPU/Host residency tracking.
- Examples: managed by the `shared_data_repository_manager` and operator ports; conversion helpers in `src/data/` and `src/include/data/data_batch_utils.hpp`.
- Pattern: Operators lock the batch, optionally trigger conversion, then read columns into cuDF.

**`sirius::expression` (Phase 7a PIMPL):**
- Purpose: Opaque wrapper exposed to operators / executor / plan-generator so they don't include any `duckdb/planner/expression/...` headers.
- Examples: `src/include/expression/expression.hpp` (public), `src/include/expression/expression_internal.hpp` (impl bridge), `src/expression/expression.cpp`.
- Pattern: Currently wraps `std::unique_ptr<duckdb::Expression>`; in Phase 8 of the Phase 7b plan it will hold a `std::unique_ptr<sirius::ast::node>` instead.

**`sirius::ast::node` (Phase 1 — landed):**
- Purpose: Sirius-native expression tree, designed to replace `duckdb::Expression` below the plan-generator boundary.
- Examples: `src/include/expression/ast/node.hpp`, plus per-alternative headers `reference.hpp`, `constant.hpp`, `comparison.hpp`, `conjunction.hpp`, `between.hpp`, `case_expr.hpp`, `cast.hpp`, `unary_op.hpp`, `coalesce.hpp`, `in_list.hpp`, `function_call.hpp`.
- Pattern: `struct node { std::variant<...> v; }` (struct wrapper so children can store `std::unique_ptr<node>` despite the variant requiring complete types); move-only; alternative order is ABI for `std::visit` dispatch.

**`sirius::value` (Phase 2 — landed):**
- Purpose: Type-safe constant payload for `sirius::ast::constant`, keyed on `sirius::type_id` (the Phase 6 Sirius type system).
- Examples: `src/include/expression/value.hpp`, `src/expression/value.cpp`, `test/cpp/expression/test_value.cpp`.
- Pattern: `std::variant` over named carrier structs (`null_value`, `date_value`, `timestamp_sec_value`, `timestamp_value`, `decimal_value`, plus primitive numeric/string variants) so `std::visit` distinguishes semantically distinct types that share an underlying representation.

**`sirius::function_id` (Phase 3 — landed):**
- Purpose: Closed enum of every scalar function the GPU executor handles (27 entries: 6 arithmetic, 9 string, 9 temporal, 3 struct/control).
- Examples: `src/include/expression/function_id.hpp`, `src/expression/function_id.cpp`, `test/cpp/expression/test_function_id.cpp`.
- Pattern: `enum class function_id : uint16_t`; `from_duckdb_function_name(std::string_view)` / `to_duckdb_function_name(function_id)` form the name-mapping boundary; `gpu_execute_function` switches on `function_id` instead of string-matching DuckDB function names.

**`sirius::join_condition` (Phase 7a mirror struct):**
- Purpose: Sirius-side mirror of `duckdb::JoinCondition` so join operators stop including DuckDB join planner headers.
- Examples: `src/include/expression/join_condition.hpp`, `src/expression/join_condition.cpp`.

**`duckdb::SiriusContext` (extension state):**
- Purpose: Per-DuckDB-connection root that owns every Sirius subsystem and implements `OnFinalizePrepare`.
- Examples: `src/include/sirius_context.hpp`, `src/sirius_context.cpp`.
- Pattern: `ClientContextState` subclass; `initialize(config)` / `terminate()` lifecycle; `transparent_execution_stats` counts; `InternalQueryGuard` suppresses lifecycle hooks for re-entrant internal connections (used by Iceberg metadata lookup).

## Entry Points

**Transparent execution:**
- Location: `src/transparent/sirius_optimizer_extension.cpp` (`sirius_pre_optimizer_hook`, `sirius_optimizer_hook`); `src/include/sirius_context.hpp` (`SiriusContext::OnFinalizePrepare`); `src/transparent/physical_sirius_execution.cpp` (`sirius::transparent::PhysicalSiriusExecution`).
- Triggers: Any DuckDB query, provided `~/.sirius/sirius.yaml` (or legacy `sirius.cfg`) is present and transparent execution is enabled. DuckDB's optimizer / finalize-prepare invokes the hooks automatically.
- Responsibilities: Capture the logical plan; replace the CPU physical plan with a Sirius plan when supported; record fallback stats; lazily run the Sirius engine on first `GetData` call.

**Explicit GPU execution table function:**
- Location: `src/sirius_extension.cpp` — `SiriusExtension::GPUExecutionFunction` (bound by `gpu_execution` `TableFunction` registered in `LoadInternal`); declared on `src/include/sirius_extension.hpp`.
- Triggers: `CALL gpu_execution('SELECT ...')` with optional `enable_optimizer` / `query_label` named parameters.
- Responsibilities: Re-parse and re-optimize the SQL inside the table function, generate a Sirius plan, run it through `sirius_interface`, stream the result back as a table.

**Per-connection query lifecycle:**
- Location: `src/sirius_interface.cpp` — `sirius_interface::sirius_execute_query`, `fetch_result_internal`.
- Triggers: Called from both transparent and explicit entry points.
- Responsibilities: Manage the `PendingQueryResult`/`MaterializedQueryResult` lifecycle, drive `sirius_engine` execution, finalize telemetry, surface errors and fallback signalling.

**Extension load + config:**
- Location: `src/sirius_extension.cpp` — `SiriusExtension::Load` and `LoadInternal` register the `sirius_state` `ClientContextState` factory, the `gpu_execution` table function, optional `gpu_processing` + `gpu_buffer_init` legacy shims (when `SIRIUS_ENABLE_LEGACY` is on), settings (`SET sirius_*`), and the transparent optimizer extension.
- Triggers: `LOAD 'sirius.duckdb_extension'`.
- Responsibilities: Initialize per-connection state, parse YAML config, set up telemetry, install hooks.

**Legacy `gpu_processing`:**
- Location: `src/sirius_extension.cpp` (`gpu_processing` and `gpu_buffer_init` table functions, only registered when built with `-DENABLE_LEGACY_SIRIUS=ON`); execution under `src/legacy/gpu_executor.cpp`, plan generation under `src/legacy/gpu_physical_plan_generator.cpp`, operators under `src/legacy/operator/`, kernels under `src/legacy/cuda/`.
- Triggers: `CALL gpu_buffer_init(...)` then `CALL gpu_processing('SELECT ...')`.
- Responsibilities: Run the deprecated single-threaded path; no new work lands here.

## Error Handling

**Strategy:** Try GPU execution; gracefully fall back to DuckDB CPU when an operator/type isn't supported, the data exceeds GPU memory even after spilling, or the libcudf int32 row-count limit (~2B) is exceeded. Transparent execution silently falls back; explicit `CALL gpu_execution(...)` returns a not-implemented error so the user sees what's unsupported.

**Patterns:**
- **Plan-time fallback:** `sirius_physical_plan_generator::create_plan()` throws / returns sentinel → `OnFinalizePrepare` leaves the DuckDB CPU plan in place and bumps `transparent_execution_stats::fallbacks`.
- **Runtime fallback (transparent):** `PhysicalSiriusExecution::GetDataInternal` catches exceptions from `sirius_interface`; if fallback is enabled, DuckDB's CPU plan is run instead.
- **Runtime fallback (explicit):** `sirius_interface` with `fallback_enabled` re-runs the query via DuckDB's CPU executor; otherwise the error propagates.
- **OOM retry:** `gpu_pipeline_executor` catches `oom_reschedule_exception` (`src/include/pipeline/oom_reschedule_exception.hpp`) — up to 10 retries with 5ms back-off after triggering a downgrade.
- **Other execution errors:** propagated to `completion_handler::report_error()`, which drains the task queue and signals the main thread's future.
- **Legacy fallback list:** `src/legacy/fallback.cpp` (`duckdb::FallbackChecker`) — only used by the legacy path. Phase 7b intentionally does not touch this file (see `.planning/STATE.md`).

## Cross-Cutting Concerns

**Logging:** spdlog via `src/include/log/logging.hpp` (`SIRIUS_LOG_*` macros). Env vars: `SIRIUS_LOG_DIR` (defaults to `${CMAKE_BINARY_DIR}/log`), `SIRIUS_LOG_LEVEL` (`trace` / `debug` / `info` / `warn` / `error`).

**Telemetry:** quent-backed in `src/telemetry/telemetry_context.cpp` and `src/include/telemetry/telemetry_context.hpp`. `sirius_engine` creates a per-query handle (`quent::query::create`) plus a query-group observer; `sirius_interface` owns the per-connection telemetry context.

**Validation:** Type checks during plan generation (`ast_supported_types.hpp`, `helper/logical_type.hpp`); row-count assertions in operators; reservation invariants in the memory manager; pre-commit hooks (clang-format, clang-tidy `WarningsAsErrors: '*'`, codespell, cmake-format) keep the boundary stable.

**Exceptions:** Sirius-native exceptions in `src/include/sirius/exception.hpp` (`sirius::not_implemented_exception`, `invalid_input_exception`, etc.) — preferred over `duckdb::Exception` below the plan-builder boundary.

**Expression evaluation dispatch:** `gpu_expression_executor::execute()` → one of `MATERIALIZE` / `AST_INTERPRET` / `AST_JIT` (per `expression_executor_strategy`) → per-node specialization → cuDF / `cudf::ast::tree` / cuJIT.

**Memory barrier semantics:** Repository ports carry a `MemoryBarrierType` (`FULL`, `PARTIAL`, `PIPELINE`) that controls when downstream consumers are released and how completion propagates through the pipeline graph.

**Concurrency primitives:** `src/include/exec/` (`bounded_thread_pool.hpp`, `channel.hpp`, `inspectable_mpsc.hpp`, `interruptible_mpmc.hpp`, `scoped_dispatcher.hpp`, `thread_pool.hpp`) are the shared async building blocks used by the scheduler, task creator, scan executor, and downgrade executor.

---

*Architecture analysis: 2026-05-20 — refresh of 2026-04-24 map; reflects Phase 1-3 completion of the Phase 7b expression-framework milestone (`sirius::ast::node`, `sirius::value`, `sirius::function_id` landed via PRs #707 / #715 / #716). The legacy `gpu_processing` path is frozen and remains gated by `-DENABLE_LEGACY_SIRIUS=ON`. Phase 4 — `sirius::ast::from_duckdb` translator (additive) — is the next change.*
