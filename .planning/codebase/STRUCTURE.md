# Codebase Structure

**Analysis Date:** 2026-05-20

> Cross-link: see `.planning/codebase/ARCHITECTURE.md` for the layer / data-flow view of these directories, and `docs/super-sirius/README.md` for the canonical narrative architecture docs.

## Directory Layout

```
sirius_expression_framework/
├── CMakeLists.txt                     # Main build; includes src/legacy/CMakeLists.txt when -DENABLE_LEGACY_SIRIUS=ON
├── Makefile                           # DuckDB extension build wrapper (includes extension-ci-tools)
├── extension_config.cmake             # Which extensions to bundle (sirius, json, tpcds, tpch, parquet, icu)
├── pixi.toml / pixi.lock              # Pixi env (CUDA 13+, cuDF, RMM, cuCascade)
├── pre-commit-config.yaml             # clang-format, clang-tidy, codespell, cmake-format
├── CLAUDE.md / CONTRIBUTING.md        # Project rules
│
├── src/                               # Super Sirius source (namespace sirius); top-level files own the extension shim
│   ├── sirius_extension.cpp           # Extension load, table functions (gpu_execution + legacy gpu_processing/gpu_buffer_init)
│   ├── sirius_interface.cpp           # Query interface (pending/execute/fetch)
│   ├── sirius_engine.cpp              # Pipeline building + execution orchestration
│   ├── sirius_context.cpp             # SiriusContext (ClientContextState) subsystem lifecycle
│   ├── sirius_config.cpp              # YAML / cfg config loader (sirius_config struct)
│   ├── config.cpp                     # Runtime config knobs + SET handlers
│   ├── pin_table.cpp                  # `pin_table` helper used by the extension
│   ├── debug_utils.cpp                # Plan / pipeline debug printers
│   │
│   ├── transparent/                   # DuckDB hook integration (transparent execution)
│   │   ├── sirius_optimizer_extension.cpp   # Pre/post optimizer hooks: capture logical plan
│   │   └── physical_sirius_execution.cpp    # PhysicalSiriusExecution: plan-replacement wrapper
│   │
│   ├── planner/                       # Logical-to-physical translation (ONLY place that touches duckdb::Bound*Expression)
│   │   ├── sirius_physical_plan_generator.cpp   # Entry point: create_plan()
│   │   ├── query.cpp                  # Per-query planning context
│   │   ├── sirius_plan_aggregate.cpp           # GROUP BY / aggregation
│   │   ├── sirius_plan_column_data_get.cpp     # COLUMN_DATA_GET source
│   │   ├── sirius_plan_comparison_join.cpp     # Hash / nested-loop joins
│   │   ├── sirius_plan_cte.cpp                 # CTEs
│   │   ├── sirius_plan_delim_get.cpp           # Delim get (delim-join right side)
│   │   ├── sirius_plan_delim_join.cpp          # Delim joins
│   │   ├── sirius_plan_dummy_scan.cpp          # DUMMY_SCAN
│   │   ├── sirius_plan_empty_result.cpp        # EMPTY_RESULT
│   │   ├── sirius_plan_expression_get.cpp      # EXPRESSION_GET
│   │   ├── sirius_plan_filter.cpp              # WHERE / filter
│   │   ├── sirius_plan_get.cpp                 # Generic table get
│   │   ├── sirius_plan_limit.cpp               # LIMIT
│   │   ├── sirius_plan_order.cpp               # ORDER BY
│   │   ├── sirius_plan_projection.cpp          # SELECT columns
│   │   ├── sirius_plan_recursive_cte.cpp       # Recursive CTE
│   │   └── sirius_plan_top_n.cpp               # TOP-N
│   │
│   ├── pipeline/                      # Pipeline construction and execution
│   │   ├── sirius_meta_pipeline.cpp            # Meta-pipeline builder (build + ready)
│   │   ├── sirius_pipeline_converter.cpp       # Splits meta-pipelines into multi-pipeline graph
│   │   ├── sirius_pipeline.cpp                 # Executable pipeline (source + ops + sink + ports)
│   │   ├── repository_wiring_materializer.cpp  # Wires repositories between split pipelines
│   │   ├── gpu_pipeline_executor.cpp           # Per-GPU manager + worker pool
│   │   ├── gpu_pipeline_task.cpp               # GPU pipeline task wrapper
│   │   ├── task_scheduler.cpp                  # Top-level orchestrator + completion handler
│   │   ├── task_request.cpp                    # Task-request protocol
│   │   └── sirius_plan_printer.cpp             # EXPLAIN-style plan visualization
│   │
│   ├── creator/                       # Task creation + scheduling
│   │   └── task_creator.cpp           # Hint-chain following + per-operator readiness
│   │
│   ├── op/                            # Physical operators (GPU-native, namespace sirius::op)
│   │   ├── sirius_physical_operator.cpp         # Base class implementation
│   │   ├── sirius_physical_operator_type.cpp    # SiriusPhysicalOperatorType enum + name
│   │   ├── sirius_dynamic_filter.cpp            # Runtime dynamic-filter shared state
│   │   ├── sirius_physical_hash_join.cpp        # Hash join (inner/left/right/outer/semi/anti/mark)
│   │   ├── sirius_physical_nested_loop_join.cpp # Nested-loop join
│   │   ├── sirius_physical_delim_join.cpp       # Delim (delimiter) join
│   │   ├── sirius_physical_grouped_aggregate.cpp / _merge.cpp
│   │   ├── sirius_physical_ungrouped_aggregate.cpp / _merge.cpp
│   │   ├── sirius_physical_order.cpp            # ORDER BY (multi-pipeline)
│   │   ├── sirius_physical_merge_sort.cpp       # Merge phase of distributed sort
│   │   ├── sirius_physical_sort_partition.cpp   # Sort partitioning
│   │   ├── sirius_physical_sort_sample.cpp      # Sort sample (range partition keys)
│   │   ├── sirius_physical_top_n.cpp / sirius_physical_top_n_merge (header-only, declared in src/include/op)
│   │   ├── sirius_physical_filter.cpp           # WHERE
│   │   ├── sirius_physical_projection.cpp       # SELECT columns
│   │   ├── sirius_physical_limit.cpp            # LIMIT
│   │   ├── sirius_physical_concat.cpp           # Row concatenation
│   │   ├── sirius_physical_partition.cpp        # Hash partition
│   │   ├── sirius_physical_partition_consumer_operator.cpp  # Partition-consumer (one per downstream split)
│   │   ├── sirius_physical_result_collector.cpp # Terminal sink → materialized result
│   │   ├── sirius_physical_cte.cpp              # CTE materialization + read
│   │   ├── sirius_physical_table_scan.cpp       # Generic table scan op
│   │   ├── sirius_physical_parquet_scan.cpp     # Parquet scan
│   │   ├── sirius_physical_iceberg_scan.cpp     # Iceberg scan
│   │   ├── sirius_physical_duckdb_scan.cpp      # Pass-through DuckDB scan
│   │   ├── sirius_physical_column_data_scan.cpp # In-memory ColumnDataCollection scan
│   │   ├── sirius_physical_cpu_source.cpp       # CPU-side source (bridge from non-GPU tasks)
│   │   ├── sirius_physical_dummy_scan.cpp       # DUMMY_SCAN
│   │   ├── sirius_physical_empty_result.cpp     # EMPTY_RESULT
│   │   │
│   │   ├── scan/                       # Scan task execution + decoders
│   │   │   ├── duckdb_scan_executor.cpp        # Scan-executor thread pool
│   │   │   ├── duckdb_scan_task.cpp            # DuckDB scan task
│   │   │   ├── duckdb_native_metadata.cpp      # Native DuckDB metadata reader
│   │   │   ├── parquet_scan_task.cpp           # Parquet scan task
│   │   │   ├── parquet_schema_mapping.cpp      # Parquet → cuDF schema mapping
│   │   │   ├── iceberg_scan_task.cpp           # Iceberg scan task
│   │   │   ├── iceberg_avro_reader.cpp         # Iceberg manifest avro reader
│   │   │   ├── iceberg_delete_pipeline.cpp     # Equality / positional delete pipeline
│   │   │   ├── iceberg_metadata_reader.cpp     # Iceberg snapshot/manifest reader
│   │   │   ├── equality_delete_filter.cpp / equality_delete_mask.cu
│   │   │   ├── positional_delete_filter.cpp
│   │   │   ├── puffin_reader.cpp               # Puffin statistics blob reader
│   │   │   ├── hive_partition.cpp              # Hive partition extraction
│   │   │   ├── cached_ranges.cpp               # Per-task cache range tracking
│   │   │   ├── cpu_source_task.cpp             # CPU-source task
│   │   │   ├── prefetched_data_source.cpp      # Prefetch manager
│   │   │   ├── scan_plan.cpp                   # Scan-plan helper
│   │   │   ├── scan_utils.cpp                  # Misc helpers
│   │   │   └── sirius_gpu_parquet_scan_operator.cpp  # GPU-native Parquet scan operator
│   │   │
│   │   ├── aggregate/                  # Grouping / aggregation helpers
│   │   │   ├── aggregate_op_util.cpp
│   │   │   └── gpu_aggregate_impl.cpp
│   │   ├── merge/                      # Merge helpers (sort/aggregate merges)
│   │   │   └── gpu_merge_impl.cpp
│   │   ├── order/                      # Sort helpers
│   │   │   └── gpu_order_impl.cpp
│   │   ├── partition/                  # Partition helpers
│   │   │   └── gpu_partition_impl.cpp
│   │   └── result/
│   │       └── host_table_chunk_reader.cpp     # Host-side reader for the result collector
│   │
│   ├── expression/                    # Sirius-native expression IR (Phase 7a wrapper + Phase 1-3 AST scaffold)
│   │   ├── expression.cpp             # sirius::expression PIMPL impl (over duckdb::Expression today)
│   │   ├── join_condition.cpp         # sirius::join_condition mirror struct
│   │   ├── value.cpp                  # sirius::value variant impl (Phase 2)
│   │   └── function_id.cpp            # sirius::function_id name mapping (Phase 3)
│   │
│   ├── expression_executor/           # GPU expression evaluation (consumes sirius::expression today)
│   │   ├── expression_executor_strategy.cpp     # MATERIALIZE / AST_INTERPRET / AST_JIT selection
│   │   ├── gpu_expression_executor.cpp          # Main dispatcher
│   │   ├── gpu_expression_translator.cpp        # DuckDB → cuDF AST bridge (consumes duckdb::Expression today)
│   │   ├── specializations/                     # Per-node GPU implementations
│   │   │   ├── gpu_execute_between.cpp
│   │   │   ├── gpu_execute_case.cpp
│   │   │   ├── gpu_execute_cast.cpp
│   │   │   ├── gpu_execute_comparison.cpp
│   │   │   ├── gpu_execute_conjunction.cpp
│   │   │   ├── gpu_execute_constant.cpp
│   │   │   ├── gpu_execute_function.cpp         # Now switches on sirius::function_id
│   │   │   ├── gpu_execute_operator.cpp
│   │   │   └── gpu_execute_reference.cpp
│   │   └── regex/
│   │       └── regex_playground.cpp             # Regex helper (cuDF strings::replace_re)
│   │
│   ├── memory/                        # GPU/Host/Disk memory + reservation manager
│   │   ├── sirius_memory_reservation_manager.cpp
│   │   └── defragmenter_oom_policy.cpp
│   │
│   ├── downgrade/                     # GPU → Host spill executor
│   │   └── downgrade_executor.cpp
│   │
│   ├── data/                          # Data batch + parquet representation helpers
│   │   ├── host_parquet_representation.cpp
│   │   └── host_parquet_representation_converters.cpp
│   │
│   ├── scan_manager/                  # Split-aware scan management
│   │   ├── sirius_scan_manager.cpp
│   │   ├── cached_split_provider.cpp
│   │   ├── parquet_split_provider.cpp
│   │   ├── split_connector.cpp
│   │   └── split_provider.cpp
│   │
│   ├── io/                            # IO subsystem (local + S3 + io_uring)
│   │   ├── sirius_datasource.cpp
│   │   ├── datasource_factory.cpp
│   │   ├── io_context.cpp
│   │   ├── admission_control.cpp
│   │   ├── prefetching_cache.cpp
│   │   ├── uri_parser.cpp
│   │   ├── s3/
│   │   │   ├── sigv4.cpp
│   │   │   └── sirius_sigv4_credential_provider.cpp
│   │   └── uring/
│   │       ├── uring_ioctx.cpp
│   │       └── uring_reactor.cpp
│   │
│   ├── cuda/                          # GPU kernels for the active path
│   │   └── scan/
│   │       ├── gpu_decode_alp.cu
│   │       ├── gpu_decode_bitpacking.cu
│   │       ├── gpu_decode_rle.cu
│   │       ├── gpu_decode_strings.cu
│   │       └── gpu_native_decode.cu
│   │
│   ├── telemetry/                     # quent-backed telemetry
│   │   └── telemetry_context.cpp
│   │
│   ├── helper/                        # Cross-cutting type helpers
│   │   └── type_conversions.cpp
│   │
│   ├── parallel/                      # Parallel task executor
│   │   └── task_executor.cpp
│   │
│   ├── util/                          # Misc utilities
│   │   ├── segfault_backtrace_handler.cpp
│   │   └── stream_check_wrapper.cpp
│   │
│   ├── include/                       # Public headers (mirror of src/, namespace sirius)
│   │   ├── sirius_extension.hpp / sirius_interface.hpp / sirius_engine.hpp
│   │   ├── sirius_context.hpp                  # SiriusContext (ClientContextState)
│   │   ├── sirius_config.hpp / config.hpp / debug_utils.hpp / pin_table.hpp
│   │   ├── utils.hpp / yaml_reader.hpp / task_completion.hpp
│   │   ├── sirius/
│   │   │   └── exception.hpp                   # Sirius-native exception hierarchy
│   │   ├── expression/                         # Expression IR headers (CRITICAL — Phase 7b territory)
│   │   │   ├── expression.hpp                  # Phase 7a PIMPL public header
│   │   │   ├── expression_internal.hpp         # Phase 7a PIMPL impl bridge (unwrap / release)
│   │   │   ├── value.hpp                       # sirius::value variant (Phase 2)
│   │   │   ├── function_id.hpp                 # sirius::function_id closed enum (Phase 3)
│   │   │   ├── join_condition.hpp              # Phase 7a join_condition mirror struct
│   │   │   └── ast/                            # Phase 1 AST scaffold (sirius::ast::node)
│   │   │       ├── node.hpp                    # Struct wrapping std::variant<...>
│   │   │       ├── reference.hpp
│   │   │       ├── constant.hpp
│   │   │       ├── comparison.hpp
│   │   │       ├── conjunction.hpp
│   │   │       ├── between.hpp
│   │   │       ├── case_expr.hpp
│   │   │       ├── cast.hpp
│   │   │       ├── unary_op.hpp
│   │   │       ├── coalesce.hpp
│   │   │       ├── in_list.hpp
│   │   │       └── function_call.hpp
│   │   ├── expression_executor/
│   │   │   ├── gpu_expression_executor.hpp
│   │   │   ├── gpu_expression_translator_internal.hpp
│   │   │   ├── expression_executor_strategy.hpp
│   │   │   ├── ast_supported_types.hpp         # sirius::type_id-keyed AST allowlist (Phase 3)
│   │   │   └── regex/                          # Regex helper headers
│   │   ├── planner/
│   │   │   ├── sirius_physical_plan_generator.hpp
│   │   │   └── query.hpp
│   │   ├── pipeline/
│   │   │   ├── sirius_pipeline.hpp / sirius_meta_pipeline.hpp / sirius_pipeline_converter.hpp
│   │   │   ├── sirius_pipeline_itask.hpp / sirius_pipeline_task_states.hpp
│   │   │   ├── gpu_pipeline_executor.hpp / gpu_pipeline_task.hpp
│   │   │   ├── task_scheduler.hpp / task_request.hpp
│   │   │   ├── completion_handler.hpp
│   │   │   ├── oom_reschedule_exception.hpp
│   │   │   ├── pipeline_build_context.hpp / pipeline_memory_history.hpp
│   │   │   ├── repository_wiring.hpp
│   │   │   ├── sirius_plan_printer.hpp
│   │   │   └── batch_lock_utils.hpp
│   │   ├── op/
│   │   │   ├── sirius_physical_operator.hpp / sirius_physical_operator_type.hpp
│   │   │   ├── sirius_dynamic_filter.hpp
│   │   │   ├── sirius_physical_*.hpp           # one per operator (mirrors src/op/*.cpp)
│   │   │   ├── scan/                           # mirror of src/op/scan/
│   │   │   ├── aggregate/ / merge/ / order/ / partition/ / result/
│   │   ├── creator/task_creator.hpp
│   │   ├── downgrade/downgrade_executor.hpp
│   │   ├── memory/
│   │   │   ├── sirius_memory_reservation_manager.hpp
│   │   │   ├── defragmenter_oom_policy.hpp
│   │   │   ├── host_table_utils.hpp
│   │   │   ├── multiple_blocks_allocation_accessor.hpp
│   │   │   └── resource_ref_utils.hpp
│   │   ├── data/
│   │   │   ├── cached_data_representation.hpp
│   │   │   ├── convertible_data.hpp / convertible_data_batch.hpp / convertible_gpu_pipeline_task.hpp
│   │   │   ├── data_batch_utils.hpp
│   │   │   ├── host_parquet_representation.hpp / host_parquet_representation_converters.hpp
│   │   │   └── sirius_converter_registry.hpp
│   │   ├── scan_manager/
│   │   │   ├── sirius_scan_manager.hpp / split_provider.hpp / split_connector.hpp
│   │   │   ├── cached_split_provider.hpp / parquet_split_provider.hpp / parquet_metadata.hpp
│   │   ├── io/
│   │   │   ├── sirius_datasource.hpp / datasource_factory.hpp / io_context.hpp / io_errors.hpp
│   │   │   ├── admission_control.hpp / prefetching_cache.hpp / templated_ioctx.hpp
│   │   │   ├── object_store_config.hpp / uri_parser.hpp / io_utils.hpp / types.hpp
│   │   │   ├── s3/                             # credential providers + sigv4
│   │   │   └── uring/                          # io_uring reactor + ioctx
│   │   ├── exec/                               # Header-only concurrency primitives
│   │   │   ├── bounded_thread_pool.hpp / thread_pool.hpp
│   │   │   ├── channel.hpp / inspectable_mpsc.hpp / interruptible_mpmc.hpp
│   │   │   ├── scoped_dispatcher.hpp / config.hpp
│   │   ├── telemetry/telemetry_context.hpp
│   │   ├── transparent/
│   │   │   ├── physical_sirius_execution.hpp
│   │   │   └── sirius_optimizer_extension.hpp
│   │   ├── helper/                             # Type / common helpers
│   │   │   ├── common.h / helper.hpp / logical_type.hpp
│   │   │   ├── type_conversions.hpp / types.hpp / utils.hpp
│   │   ├── common/                             # optional_ptr, reference_map
│   │   ├── log/logging.hpp
│   │   ├── parallel/                           # task / task_executor / config
│   │   ├── cudf/cudf_utils.hpp
│   │   ├── util/                               # segfault_backtrace, stream_check_wrapper
│   │   └── legacy/                             # Headers for the frozen path (built only when ENABLE_LEGACY_SIRIUS=ON)
│   │
│   └── legacy/                        # Old gpu_processing path (namespace duckdb), gated by -DENABLE_LEGACY_SIRIUS=ON
│       ├── CMakeLists.txt             # Conditional legacy build
│       ├── gpu_executor.cpp / gpu_buffer_manager.cpp / gpu_context.cpp
│       ├── gpu_physical_plan_generator.cpp / gpu_physical_operator.cpp
│       ├── gpu_meta_pipeline.cpp / gpu_pipeline.cpp / gpu_query_result.cpp / gpu_columns.cpp
│       ├── cpu_cache.cpp              # Host-side cache
│       ├── fallback.cpp               # Legacy FallbackChecker (Phase 7b never touches this)
│       ├── operator/                  # Legacy operators (gpu_physical_*.cpp)
│       ├── plan/                      # Legacy plan builders (gpu_plan_*.cpp)
│       ├── expression_executor/
│       │   ├── gpu_expression_executor.cpp / gpu_expression_executor_state.cpp
│       │   └── specializations/       # Legacy per-node specializations
│       └── cuda/                      # Legacy CUDA kernels
│           ├── operator/              # arbitrary_expression, comparison_expression, hash_join_*, materialize, nested_loop_join, strings_matching, strlen_from_offsets, substring, etc.
│           ├── cudf/                  # cudf_aggregate / cudf_groupby / cudf_join / cudf_orderby / cudf_duplicate_elimination / cudf_utils
│           ├── expression_executor/   # gpu_dispatch_materialize / gpu_dispatch_select / gpu_dispatch_string
│           ├── allocator.cu / communication.cu / print.cu / utils.cu
│
├── test/                              # Tests
│   ├── cpp/                           # C++ unit tests (Catch2)
│   │   ├── unittest.cpp               # Test runner
│   │   ├── utils/                     # Test utilities (sirius_test_env, data_utils, validation utils)
│   │   ├── config/                    # Config + context + object-store-config tests
│   │   ├── data/                      # Convertible data + host parquet rep tests
│   │   ├── debug/                     # debug_utils tests
│   │   ├── downgrade/                 # Spill executor tests
│   │   ├── exec/                      # Thread pool + MPSC/MPMC tests
│   │   ├── expression/                # AST + value + function_id + expression PIMPL + join_condition tests
│   │   │   ├── test_ast_scaffold.cpp          # Phase 1 (sirius::ast::node)
│   │   │   ├── test_value.cpp                 # Phase 2 (sirius::value)
│   │   │   ├── test_function_id.cpp           # Phase 3 (sirius::function_id)
│   │   │   ├── test_expression.cpp            # Phase 7a PIMPL wrapper
│   │   │   └── test_join_condition.cpp        # Phase 7a join_condition mirror
│   │   ├── expression_executor/       # Executor + translator tests (the 70/70 suite Phase 7b protects)
│   │   │   ├── test_gpu_expression_executor.cpp
│   │   │   └── test_gpu_expression_translator.cpp
│   │   ├── helper/ / parallel/
│   │   ├── integration/               # End-to-end + transparent-execution tests
│   │   │   ├── test_gpu_execution_multi_format.cpp
│   │   │   ├── test_gpu_execution_tpch.cpp
│   │   │   ├── test_tpcds_plan_translation.cpp
│   │   │   ├── test_transparent_execution.cpp
│   │   │   ├── integration.yaml / data/
│   │   ├── io/                        # URI parser + datasource factory + s3 tests
│   │   ├── memory/ / memory_management/   # Reservation manager, host_table_utils, cpu_cache
│   │   ├── operator/                  # Per-operator tests (filter, projection, limit, top_n, order, merge_sort, partition, ungrouped agg, concat, result collector, table scan, mark join, dynamic filter, peak memory)
│   │   ├── pipeline/                  # Executor + task scheduler + OOM reschedule + repository wiring materializer + plan printer + history
│   │   ├── planner/                   # Distinct-hash-join detection etc.
│   │   ├── scan/                      # GPU decoders + native walker + parquet scan/schema + split provider + column builder + scan executor
│   │
│   ├── sql/                           # SQL logic tests (SQLLogic format)
│   ├── answers/                       # Expected query results
│   ├── tpch_mod_sirius.test           # End-to-end TPC-H driver
│   ├── tpch_performance/ / tpcds_performance/   # Benchmark scripts + queries
│   └── io/                            # SQL-layer IO test fixtures
│
├── docs/
│   ├── super-sirius/                  # Canonical architecture reference (see README.md for index)
│   │   ├── README.md                  # Doc index + reading order
│   │   ├── architecture-overview.md / execution-flow.md
│   │   ├── physical-plan-generation.md / operators.md
│   │   ├── expression-executor.md / pipeline-execution.md / task-creator.md
│   │   ├── scan.md / memory-management.md / data-management.md
│   │   ├── configuration.md / optimizations.md / dynamic-filters.md
│   ├── README.md / DEVELOPMENT.md / UPDATING.md / glossary.md
│   ├── gpu_execution.md / gpu_processing.md   # User-facing per-entry-point docs
│
├── .claude/skills/                    # Local Claude skills (build-errors, dataset-manager, module-context, module-discover, optimization-advisor, profile-analyzer, skill-creator, update-docs)
├── .planning/                         # GSD planning artifacts (PROJECT.md, STATE.md, ROADMAP.md, phases/, codebase/)
├── tools/                             # parse_pipeline_log.py and other debug helpers
├── scripts/                           # Misc shell scripts
├── duckdb/ / duckdb-python/ / substrait/ / vcpkg/ / extension-ci-tools/ / cucascade/ # Submodules
├── cmake/ / docker/ / utils/
├── pixi.toml / pixi.lock              # Pixi env definition
└── CMakeLists.txt                     # Root CMake (CUDA 13+, C++20, GPU arches 75-120)
```

## Directory Purposes

**`src/` (top level)**
- Purpose: Super Sirius core; the loose `.cpp` files at the top own the extension shim (`sirius_extension.cpp`), per-connection interface (`sirius_interface.cpp`), engine orchestration (`sirius_engine.cpp`), context lifecycle (`sirius_context.cpp`), config (`sirius_config.cpp` + `config.cpp`), and a handful of helpers (`pin_table.cpp`, `debug_utils.cpp`).
- Contains: everything in `namespace sirius` outside `legacy/`.
- Key files: `sirius_extension.cpp`, `sirius_engine.cpp`, `sirius_interface.cpp`, `sirius_context.cpp`.

**`src/transparent/`**
- Purpose: DuckDB hook integration for transparent execution (no `CALL gpu_execution` required).
- Key files: `sirius_optimizer_extension.cpp` (pre/post optimizer hooks), `physical_sirius_execution.cpp` (`PhysicalSiriusExecution` plan-replacement wrapper).

**`src/planner/`**
- Purpose: Logical-to-physical translation; the **only** place that consumes `duckdb::Bound*Expression`. Every other layer below it consumes Sirius-native expression types.
- Key file: `sirius_physical_plan_generator.cpp` (entry point `create_plan()`). Each `sirius_plan_<construct>.cpp` builds the matching operator subtree.

**`src/pipeline/`**
- Purpose: Pipeline representation, building, multi-pipeline splitting, and execution.
- Key files: `sirius_meta_pipeline.cpp`, `sirius_pipeline_converter.cpp`, `sirius_pipeline.cpp`, `repository_wiring_materializer.cpp`, `gpu_pipeline_executor.cpp`, `gpu_pipeline_task.cpp`, `task_scheduler.cpp`, `task_request.cpp`.

**`src/creator/`**
- Purpose: Decide which operator is next-ready after each task completes.
- Key file: `task_creator.cpp`.

**`src/op/`**
- Purpose: All Super Sirius physical operators (≈30+ files) plus per-operator-category helpers in `scan/`, `aggregate/`, `merge/`, `order/`, `partition/`, `result/`. Each operator pairs with a matching header under `src/include/op/`.

**`src/op/scan/`**
- Purpose: Scan task execution + decoders + Iceberg/Parquet/DuckDB-native readers.
- Key files: `duckdb_scan_executor.cpp`, `parquet_scan_task.cpp`, `iceberg_scan_task.cpp`, `iceberg_delete_pipeline.cpp`, `prefetched_data_source.cpp`, `puffin_reader.cpp`, `scan_plan.cpp`, `equality_delete_mask.cu`, `sirius_gpu_parquet_scan_operator.cpp`.

**`src/op/aggregate/` `merge/` `order/` `partition/` `result/`**
- Purpose: Per-operator-category GPU helpers (e.g. `gpu_aggregate_impl.cpp`, `gpu_merge_impl.cpp`, `gpu_order_impl.cpp`, `gpu_partition_impl.cpp`, `host_table_chunk_reader.cpp`).

**`src/expression/`** *(new vs the 2026-04-24 map)*
- Purpose: Sirius-native expression IR — both the Phase 7a `sirius::expression` PIMPL wrapper *and* the Phase 1-3 scaffold (`sirius::value`, `sirius::function_id`, and forthcoming `sirius::ast::from_duckdb`).
- Key files: `expression.cpp`, `join_condition.cpp`, `value.cpp`, `function_id.cpp`.

**`src/expression_executor/`**
- Purpose: GPU expression evaluation.
- Key files: `gpu_expression_executor.cpp` (dispatcher), `gpu_expression_translator.cpp` (cuDF AST bridge), `expression_executor_strategy.cpp`, per-node specializations under `specializations/`, `regex/regex_playground.cpp`.

**`src/memory/` + `src/downgrade/`**
- Purpose: GPU/Host/Disk reservations (`sirius_memory_reservation_manager.cpp`, `defragmenter_oom_policy.cpp`) and the monitor / worker pool that spills GPU → Host (`downgrade_executor.cpp`).

**`src/data/`**
- Purpose: Host Parquet representation + converters used by the convertible-data pipeline. Most data types live in headers under `src/include/data/` (`convertible_data*`, `data_batch_utils`, `sirius_converter_registry`).

**`src/scan_manager/`** *(new vs the 2026-04-24 map)*
- Purpose: Split-aware scan management (parquet split provider, cached split provider, connectors), consumed by the planner + scan executor.

**`src/io/`** *(new vs the 2026-04-24 map)*
- Purpose: IO subsystem — datasource factory + io context + admission control + prefetching cache + URI parser + S3 (sigv4 + credential provider) + Linux io_uring reactor/ioctx.

**`src/cuda/`**
- Purpose: GPU kernels for the active path. Currently scoped to scan decoders (`gpu_decode_alp.cu`, `gpu_decode_bitpacking.cu`, `gpu_decode_rle.cu`, `gpu_decode_strings.cu`, `gpu_native_decode.cu`).

**`src/telemetry/`** *(new vs the 2026-04-24 map)*
- Purpose: quent-backed telemetry (`telemetry_context.cpp`); per-query handle + query-group observer plumbing.

**`src/helper/` + `src/util/` + `src/parallel/`**
- Purpose: Cross-cutting utilities (`type_conversions.cpp`, `segfault_backtrace_handler.cpp`, `stream_check_wrapper.cpp`, `task_executor.cpp`).

**`src/legacy/` (gated, namespace `duckdb`)**
- Purpose: Old `gpu_processing` path. Frozen. Not touched by Phase 7b commits.
- Build: only compiled when `-DENABLE_LEGACY_SIRIUS=ON`.
- Key files: `gpu_executor.cpp`, `gpu_buffer_manager.cpp`, `gpu_physical_plan_generator.cpp`, `cpu_cache.cpp`, `fallback.cpp` (legacy `FallbackChecker`), plus `operator/gpu_physical_*.cpp`, `plan/gpu_plan_*.cpp`, `expression_executor/*`, `cuda/*`.

**`src/include/`**
- Purpose: Public headers mirroring the `src/` layout under namespace `sirius`. Includes the `sirius/exception.hpp` family, the `expression/` IR (with `ast/` scaffold), per-subsystem `pipeline/`, `op/`, `planner/`, `creator/`, `downgrade/`, `memory/`, `data/`, `scan_manager/`, `io/`, `exec/`, `telemetry/`, `transparent/`, `helper/`, `common/`, `log/`, `parallel/`, `cudf/`, `util/`, plus the `legacy/` headers (only consumed when `-DENABLE_LEGACY_SIRIUS=ON`).

**`src/include/expression/ast/`** *(critical for Phase 7b)*
- Purpose: Sirius-native expression AST scaffold. `node.hpp` declares `struct node { std::variant<...> v; }`; each per-kind header (`reference.hpp`, `constant.hpp`, `comparison.hpp`, `conjunction.hpp`, `between.hpp`, `case_expr.hpp`, `cast.hpp`, `unary_op.hpp`, `coalesce.hpp`, `in_list.hpp`, `function_call.hpp`) defines one alternative.

**`src/include/exec/`**
- Purpose: Header-only concurrency primitives (`bounded_thread_pool.hpp`, `channel.hpp`, `inspectable_mpsc.hpp`, `interruptible_mpmc.hpp`, `scoped_dispatcher.hpp`, `thread_pool.hpp`).

**`test/cpp/`**
- Purpose: Catch2 unit tests. Organized by subsystem mirror of `src/` (config, data, debug, downgrade, exec, expression, expression_executor, helper, parallel, integration, io, memory, memory_management, operator, pipeline, planner, scan, utils). Runner: `unittest.cpp`. Test environment helpers in `utils/sirius_test_env.cpp`.

**`test/sql/` + `test/answers/` + `test/tpch_mod_sirius.test`**
- Purpose: SQL logic tests + expected outputs.
- Note: `tpch-sirius.test` exercises the legacy `gpu_processing` path and is therefore frozen for Phase 7b commits (`.planning/STATE.md`).

**`test/tpch_performance/` + `test/tpcds_performance/`**
- Purpose: Performance benchmarking scripts (`generate_test_data.py`, `performance_test.py`) plus per-query SQL.

**`docs/super-sirius/`**
- Purpose: Canonical architecture documentation. The map files in `.planning/codebase/` link to these.

**`.planning/`**
- Purpose: GSD planning artifacts — `PROJECT.md`, `STATE.md`, `ROADMAP.md`, per-phase folders under `phases/`, and this codebase map (`codebase/`).

## Key File Locations

**Entry points:**
- `src/sirius_extension.cpp` — `SiriusExtension::LoadInternal()` registers the `sirius_state` factory, the `gpu_execution` table function, settings, and (when legacy is enabled) `gpu_buffer_init` + `gpu_processing` shims.
- `src/transparent/sirius_optimizer_extension.cpp` — pre/post optimizer hooks (`sirius_pre_optimizer_hook`, `sirius_optimizer_hook`).
- `src/transparent/physical_sirius_execution.cpp` — `sirius::transparent::PhysicalSiriusExecution::GetDataInternal` (transparent execution entry).
- `src/sirius_interface.cpp` — `sirius_interface::sirius_execute_query`, `fetch_result_internal`.

**Configuration:**
- `src/config.cpp` / `src/include/config.hpp` — runtime knobs.
- `src/sirius_config.cpp` / `src/include/sirius_config.hpp` — YAML / cfg loader + sirius_config struct.
- `src/include/sirius_context.hpp` — `duckdb::SiriusContext` (owns every subsystem).

**Core orchestration:**
- `src/sirius_engine.cpp` — pipeline building + execute.
- `src/planner/sirius_physical_plan_generator.cpp` — DuckDB→Sirius plan translation entry point.
- `src/pipeline/sirius_meta_pipeline.cpp`, `src/pipeline/sirius_pipeline_converter.cpp` — multi-pipeline construction.
- `src/pipeline/gpu_pipeline_executor.cpp`, `src/pipeline/task_scheduler.cpp` — execution.
- `src/creator/task_creator.cpp` — scheduling decisions.

**Expression IR (Phase 7b territory):**
- `src/include/expression/ast/node.hpp` — Sirius AST variant (Phase 1 — landed).
- `src/include/expression/value.hpp`, `src/expression/value.cpp` — `sirius::value` (Phase 2 — landed).
- `src/include/expression/function_id.hpp`, `src/expression/function_id.cpp` — closed function enum (Phase 3 — landed).
- `src/include/expression/expression.hpp` + `expression_internal.hpp`, `src/expression/expression.cpp` — Phase 7a PIMPL wrapper.
- `src/include/expression/join_condition.hpp`, `src/expression/join_condition.cpp` — Phase 7a join_condition mirror.
- `src/expression_executor/gpu_expression_executor.cpp`, `gpu_expression_translator.cpp`, `specializations/gpu_execute_*.cpp` — executor consumers.
- `src/include/expression_executor/ast_supported_types.hpp` — `sirius::type_id`-keyed AST allowlist (Phase 3 outcome).

**Testing:**
- `test/cpp/unittest.cpp` — Catch2 runner.
- `test/cpp/expression/` — `test_ast_scaffold.cpp` (`[ast_scaffold]`), `test_value.cpp` (`[ast_value]`), `test_function_id.cpp` (`[ast_function_id]`), `test_expression.cpp`, `test_join_condition.cpp`.
- `test/cpp/expression_executor/test_gpu_expression_executor.cpp`, `test_gpu_expression_translator.cpp` — the 70/70 suite Phase 7b must not regress.
- `test/cpp/integration/test_transparent_execution.cpp`, `test_gpu_execution_tpch.cpp`, `test_gpu_execution_multi_format.cpp`, `test_tpcds_plan_translation.cpp` — end-to-end coverage.
- `test/sql/tpch-sirius.test` / `test/tpch_mod_sirius.test` — SQL-logic harness (`tpch-sirius.test` is legacy-path; frozen for Phase 7b).

## Naming Conventions

**Files (Super Sirius):**
- Physical operators: `sirius_physical_<construct>.cpp` (`src/op/`).
- Plan builders: `sirius_plan_<construct>.cpp` (`src/planner/`).
- Per-operator GPU helpers: `gpu_<construct>_impl.cpp` under `src/op/<category>/`.
- Scan decoders / kernels: `gpu_decode_<codec>.cu` under `src/cuda/scan/`.
- Expression-executor specializations: `gpu_execute_<node>.cpp` under `src/expression_executor/specializations/`.
- AST headers: one file per alternative under `src/include/expression/ast/` named after the node kind (`reference.hpp`, `case_expr.hpp`, `unary_op.hpp`, etc.).
- Tests: `test_<component>.cpp` mirroring the source layout under `test/cpp/`.

**Files (Legacy):**
- Operators: `gpu_physical_<construct>.cpp` under `src/legacy/operator/`.
- Plan builders: `gpu_plan_<construct>.cpp` under `src/legacy/plan/`.
- CUDA kernels: under `src/legacy/cuda/{operator,cudf,expression_executor}/`.

**Directories:**
- Super Sirius: `src/` + `src/include/` + `test/cpp/` (namespace `sirius`).
- Legacy: `src/legacy/` + `src/include/legacy/` (namespace `duckdb::gpu*`), only compiled with `-DENABLE_LEGACY_SIRIUS=ON`.
- Per-category subdirs in `src/op/` and `src/include/op/`: `scan/`, `aggregate/`, `merge/`, `order/`, `partition/`, `result/`.

**Namespaces / classes:**
- `sirius::op::sirius_physical_<name>` — physical operators (`sirius::op::sirius_physical_hash_join`, etc.).
- `sirius::planner::sirius_physical_plan_generator` — plan generator.
- `sirius::sirius_meta_pipeline`, `sirius::sirius_pipeline`, `sirius::gpu_pipeline_executor`, `sirius::gpu_pipeline_task`, `sirius::task_scheduler`, `sirius::task_creator`.
- `sirius::expression`, `sirius::join_condition` — Phase 7a wrappers.
- `sirius::ast::<node_kind>` — AST alternatives; `sirius::ast::node` wraps the variant.
- `sirius::value` (variant), `sirius::function_id` (enum).
- `sirius::transparent::PhysicalSiriusExecution`, `sirius::transparent::sirius_optimizer_extension`.
- `duckdb::SiriusContext` (intentionally in `duckdb::` because it derives from `duckdb::ClientContextState`).
- Legacy: `duckdb::GPUPhysicalPlanGenerator`, `duckdb::GPUPhysicalOperator`, `duckdb::FallbackChecker`, etc.

**Functions / methods:**
- Plan: `create_plan()` (entry), `plan_<construct>()` per builder.
- Operator: `execute()`, `sink()`, `get_next_task_hint()`, `source_order()`, `is_source()`.
- Scheduler: `start_query()`, `prepare_for_query()`, `schedule()`, `report_error()`, `mark_completed()`.
- Memory: `reserve()`, `release()`, `downgrade_to_host()`.
- Data: `publish()`, `consume()`, `lock_or_prepare_batch()`.
- Expression: `gpu_expression_executor::execute()`, `count_ast_ops()`, `from_duckdb_function_name()` / `to_duckdb_function_name()` for the function_id boundary.

## Where to Add New Code

**New Super Sirius operator:**
1. Add header `src/include/op/sirius_physical_<op_name>.hpp` (sub-folder under `op/` if the operator splits into category helpers).
2. Add impl `src/op/sirius_physical_<op_name>.cpp` and (if needed) a `src/op/<category>/gpu_<op_name>_impl.cpp` GPU helper.
3. Add CUDA kernels under `src/cuda/<category>/` only if you need new kernels — for cuDF-backed ops just call cuDF directly.
4. Add a plan builder `src/planner/sirius_plan_<construct>.cpp` (and register the builder in `sirius_physical_plan_generator.cpp`).
5. Wire pipeline-splitting rules in `src/pipeline/sirius_pipeline_converter.cpp` if the operator splits.
6. Add Catch2 tests under `test/cpp/operator/` and SQL coverage under `test/sql/`.

**New AST node:**
1. Add `src/include/expression/ast/<node>.hpp` (struct with `std::unique_ptr<node>` children where needed).
2. Append the alternative to the `std::variant` in `src/include/expression/ast/node.hpp` — **at the end** (variant index is ABI).
3. Add a translator branch in `src/expression_executor/gpu_expression_translator.cpp` and (once dual-path lands) in `from_duckdb`.
4. Add a specialization `src/expression_executor/specializations/gpu_execute_<node>.cpp` and register in `src/expression_executor/gpu_expression_executor.cpp` dispatch.
5. If the node introduces a new function, extend `sirius::function_id` in `src/include/expression/function_id.hpp` (append, never insert) and update `from_duckdb_function_name` / `to_duckdb_function_name` in `src/expression/function_id.cpp`.
6. Add tests under `test/cpp/expression/` (typed scaffold tests) and `test/cpp/expression_executor/` (executor / translator coverage).

**New scalar function:**
1. Append a `function_id` enum value (end of the matching category in `src/include/expression/function_id.hpp`).
2. Update name mappings in `src/expression/function_id.cpp`.
3. Implement dispatch in `src/expression_executor/specializations/gpu_execute_function.cpp` (the function_id switch).
4. If AST-mode capable, add to `supported_ast_functions` and the `function_type_switch_ast` lambda in `gpu_execute_function.cpp`.
5. Tests under `test/cpp/expression/test_function_id.cpp` and `test/cpp/expression_executor/test_gpu_expression_executor.cpp`.

**New constant payload variant:**
1. Add a carrier struct in `src/include/expression/value.hpp`.
2. Extend the variant + impl in `src/expression/value.cpp`.
3. Add coverage in `test/cpp/expression/test_value.cpp`.

**New scan format / data source:**
1. Add a scan task under `src/op/scan/<format>_scan_task.cpp` and header.
2. Plug into `src/op/sirius_physical_<format>_scan.cpp` + the matching `src/planner/sirius_plan_get.cpp` branch.
3. Add a split provider under `src/scan_manager/` if the format needs splits.
4. Hook the IO layer (`src/io/`) for remote storage (S3, etc.).
5. Add tests under `test/cpp/scan/`.

**New memory / executor / scheduling subsystem:**
1. Source under `src/<subsystem>/`, header under `src/include/<subsystem>/`.
2. Wire into `duckdb::SiriusContext` (`src/include/sirius_context.hpp` + `src/sirius_context.cpp` `initialize` / `terminate`).
3. Add tests under `test/cpp/<subsystem>/`.

**New utility:**
- Cross-cutting helpers: `src/helper/` or `src/util/` (with headers under `src/include/helper/`, `src/include/util/`).
- Concurrency primitives: `src/include/exec/` (header-only).
- Logging: `SIRIUS_LOG_*` macros from `src/include/log/logging.hpp`.
- Type conversion: register in `src/data/sirius_converter_registry` (registration in `src/sirius_interface.cpp`).

**Testing:**
- Unit tests: `test/cpp/<subsystem>/` (Catch2, no GPU context required for pure-logic tests; see `test/cpp/utils/sirius_test_env.cpp` for GPU-backed harnesses).
- Integration: `test/cpp/integration/`.
- SQL logic: `test/sql/` (`.test` files + answers in `test/answers/`). Note that `test/sql/tpch-sirius.test` exercises the **legacy** path — Phase 7b commits must not touch it.
- Performance: scripts in `test/tpch_performance/` and `test/tpcds_performance/`.

## Special Directories

**`src/legacy/`**
- Purpose: Old single-threaded `gpu_processing` path.
- Generated: No.
- Committed: Yes (for compatibility).
- Build: gated by `-DENABLE_LEGACY_SIRIUS=ON` (default OFF).
- Constraint: Phase 7b never touches files under here (`.planning/STATE.md`).

**`src/include/expression/ast/`**
- Purpose: Sirius-native expression AST (Phase 1 scaffold).
- Generated: No.
- Committed: Yes.
- ABI: `std::variant` alternative order is part of the public ABI — new alternatives go at the end.

**`docs/super-sirius/`**
- Purpose: Canonical architecture reference; cross-linked from the `.planning/codebase/` map.
- Audience: Anyone modifying Super Sirius.

**`build/`**
- Purpose: CMake output (object files, libraries, binaries).
- Generated: Yes (via `make`).
- Committed: No.
- Key outputs: `build/release/extension/sirius/sirius.duckdb_extension`, `build/release/extension/sirius/sirius_loadable.duckdb_extension`, `build/release/extension/sirius/test/cpp/sirius_unittest`.
- compile_commands.json: `build/release/compile_commands.json` (or `build/debug/`, `build/relwithdebinfo/`).

**`cucascade/`, `duckdb/`, `duckdb-python/`, `substrait/`, `extension-ci-tools/`, `vcpkg/`, `rust/`**
- Purpose: Git submodules — third-party libraries the build links against (cuCascade for tiered memory; DuckDB sources for the extension API; Python wrapper; Substrait; the extension-ci-tools makefiles; vcpkg port files; Rust shim).
- Generated: No.
- Committed: As submodules.

**`.planning/`**
- Purpose: GSD planning artifacts for the Phase 7b milestone — `PROJECT.md`, `STATE.md`, `ROADMAP.md`, per-phase folders under `phases/<NN-name>/`, and this codebase map under `codebase/`.

---

*Structure analysis: 2026-05-20 — refresh of 2026-04-24 map. New top-level source directories landed since the last map: `src/expression/` (Phases 2-3 IR), `src/io/` (S3 + io_uring stack), `src/scan_manager/` (split provider), `src/telemetry/`. New include subdirs: `expression/ast/` (Phase 1 scaffold), `expression/value.hpp`, `expression/function_id.hpp`, `expression_executor/ast_supported_types.hpp`, `exec/`, `io/`, `scan_manager/`, `telemetry/`, `sirius/exception.hpp`. `src/legacy/fallback.cpp` is the surviving `FallbackChecker` (no longer under `src/`); `tpch-sirius.test` exercises the legacy path and is frozen for Phase 7b commits.*
