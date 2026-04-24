# External Integrations

**Analysis Date:** 2026-05-20

## APIs & External Services

**DuckDB Core Integration:**
- DuckDB v1.5.2 (dev environment + submodule); CI distribution builds against `duckdb_version: v1.5.1` (`.github/workflows/distribution.yml`)
  - Extension entry point: `DUCKDB_CPP_EXTENSION_ENTRY(sirius, loader)` → `duckdb::LoadInternal` (`src/sirius_extension.cpp` line 1477)
  - Registered SQL table functions (`SiriusExtension::RegisterGPUFunctions`, `src/sirius_extension.cpp` lines 979–1035):
    - `gpu_execution(query, [enable_optimizer, query_label])` — Primary explicit GPU entry point (Super Sirius)
    - `sirius_set_query_label(label)` — Attaches a label to the next `gpu_execution` invocation for telemetry
    - `profiler_start()` / `profiler_stop()` — Wrap `cudaProfilerStart` / `cudaProfilerStop` for `nsys --capture-range=cudaProfilerApi`
    - `pin_table(path, tier=, name=, cols=, n_rows=)` / `unpin_table(name)` — Pre-load Parquet tables into GPU or pinned host memory via `cucascade::memory::memory_space` (`src/pin_table.cpp`)
    - Legacy `gpu_processing(query, [enable_optimizer])` and `gpu_buffer_init(cache_size, processing_size, [pinned_memory_size])` — compiled only when `SIRIUS_ENABLE_LEGACY` is defined (`ENABLE_LEGACY_SIRIUS=ON`)
  - DBConfig extension options (`SiriusExtension::InitialGPUConfigs`, `src/sirius_extension.cpp` lines 1252–1431): `gpu_execution`, `expression_executor_strategy` (`materialize` / `ast_interpret` / `ast_jit`), `use_pin_memory`, `use_pin_memory_for_caching`, `use_cudf_expr`, `use_custom_top_n`, `use_opt_table_scan`, `opt_table_scan_num_streams`, `opt_table_scan_memcpy_size`, `print_gpu_table_max_rows`, `enable_fallback_check`, `enable_duckdb_fallback`, `enable_regex_jit_impl`, `modified_pipeline`, `scan_task_batch_size`, `default_scan_task_varchar_size`, `max_sort_partition_bytes`, `hash_partition_bytes`, `concat_batch_bytes`, `max_build_hash_table_bytes`, `scan_cache_level`, `sirius_log_level`, `sirius_log_dir`, `sirius_log_flush_seconds`, `enable_quent`, `quent_output_directory`, `quent_engine_name`
  - Transparent execution (default path, `gpu_execution=true`): registered as a `duckdb::OptimizerExtension` with `pre_optimize_function = sirius::transparent::sirius_pre_optimizer_hook` and `optimize_function = sirius::transparent::sirius_optimizer_hook` (`src/sirius_extension.cpp` lines 1448–1451; impl in `src/transparent/sirius_optimizer_extension.cpp`)
  - Connection-scoped state: `duckdb::SiriusContext` registered under key `"sirius_state"` via `SiriusContextExtensionCallback` (`src/sirius_context.cpp`); replays `OnConnectionOpened` for pre-existing connections at load time

**NVIDIA RAPIDS (GPU Data Processing):**
- cuDF 26.04.x — `find_package(cudf REQUIRED CONFIG)` (`CMakeLists.txt` line 66)
  - Linked as `cudf::cudf` into both extension targets (`CMakeLists.txt` lines 332–339)
  - Headers consumed: `cudf/io/parquet.hpp`, `cudf/io/types.hpp`, `cudf/io/datasource.hpp`, plus column/table/AST headers across the operator set
  - Operators wrapping cudf primitives: `src/op/sirius_physical_hash_join.cpp`, `src/op/aggregate/gpu_aggregate_impl.cpp`, `src/op/order/gpu_order_impl.cpp`, `src/op/partition/gpu_partition_impl.cpp`, `src/op/merge/gpu_merge_impl.cpp`, `src/op/sirius_physical_top_n.cpp`, `src/op/sirius_physical_nested_loop_join.cpp`
  - Chunked Parquet reader: used by `src/pin_table.cpp` (`cudf::io::chunked_parquet_reader`) and the scan task pipeline (`src/op/scan/parquet_scan_task.cpp`)
- RMM (RAPIDS Memory Manager) — Linked as `rmm::rmm`
  - `rmm::cuda_stream` for owning streams (`src/sirius_extension.cpp`, `src/pin_table.cpp`)
  - `rmm::device_buffer` for ad-hoc GPU allocations; pooled allocators sit inside cuCascade
- libcurand-dev — Indirect dep for cuDF random sampling kernels (pixi `[feature.dev-libs.dependencies]` and `[feature.vcpkg.dependencies]`)
- cuda-nvml-dev — Provides NVML for the optional stream-check helper library (`utils/stream_check/`)

**cuCascade (Tiered Memory Management):**
- Submodule `cucascade/` built in-tree as `cucascade_static` and exported as `cuCascade::cucascade` (`CMakeLists.txt` lines 75–105)
  - All cuCascade tests/benchmarks/shared libs disabled (`CUCASCADE_BUILD_TESTS=OFF`, `CUCASCADE_BUILD_BENCHMARKS=OFF`, `CUCASCADE_BUILD_SHARED_LIBS=OFF`, `CUCASCADE_BUILD_STATIC_LIBS=ON`)
  - In the vcpkg build, `cucascade_objects`, `cucascade_static`, `cucascade_shared` targets are patched to consume `CUDA::cudart_static` (`CMakeLists.txt` lines 107–136)
- Used by:
  - `src/memory/sirius_memory_reservation_manager.cpp` — GPU memory reservations per pipeline task
  - `src/memory/defragmenter_oom_policy.cpp` — OOM remediation policy
  - `src/downgrade/downgrade_executor.cpp` — Spill from GPU → pinned host → disk
  - `src/scan_manager/sirius_scan_manager.cpp`, `src/pin_table.cpp` — User-visible table pinning across `cucascade::memory::Tier::{GPU,HOST}`
  - `src/data/host_parquet_representation.cpp`, `src/data/host_parquet_representation_converters.cpp` — Host-tier columnar representation
  - `src/data/sirius_converter_registry.hpp` (initialised by `sirius::converter_registry::initialize()` at load time) — GPU↔HOST converters

**Rust telemetry bridge (quent):**
- Static lib `telemetry_bridge` linked into both extension targets (`CMakeLists.txt` line 339)
- Source: `rust/crates/telemetry/bridge/` (CXX bridge, `cxx = "1"`) and `rust/crates/telemetry/model/`
- Pulls quent crates from `https://github.com/rapidsai/quent.git` rev `871f8f45fd35288945a13636df8cadf50f7dbcc2` (`quent-attributes`, `quent-model`, `quent-query-engine-model`, `quent-stdlib`, `quent-codegen`)
- Imported into CMake via Corrosion v0.6.1 (`rust/crates/telemetry/bridge/CMakeLists.txt`)
- Driven from `src/telemetry/telemetry_context.cpp`; exporter toggled by extension options `enable_quent`, `quent_output_directory` (NDJSON output path), `quent_engine_name`

## Data Storage

**Databases:**
- DuckDB 1.5.2 (in-process; not a remote storage system)
  - Plumbing: `duckdb::ClientContext` passed into `sirius::sirius_interface` (`src/sirius_interface.cpp`) and `sirius::planner::sirius_physical_plan_generator` (`src/planner/sirius_physical_plan_generator.cpp`)
  - Python client (optional): `duckdb-python` submodule built with `pixi run -e duckdb-python build-duckdb-python`

**File Storage:**
- Parquet (read path)
  - Scan task graph: `src/op/scan/parquet_scan_task.cpp`, `src/op/scan/scan_plan.cpp`, `src/op/scan/scan_utils.cpp`
  - Schema mapping + cudf integration: `src/op/scan/parquet_schema_mapping.cpp`
  - GPU-native decode kernels (replacing cudf decode in hot paths): `src/cuda/scan/gpu_decode_alp.cu`, `gpu_decode_bitpacking.cu`, `gpu_decode_rle.cu`, `gpu_decode_strings.cu`, `gpu_native_decode.cu`; orchestration in `src/op/sirius_physical_parquet_scan.cpp` and `src/op/scan/sirius_gpu_parquet_scan_operator.cpp`
  - Split / prefetch / cache pipeline: `src/scan_manager/sirius_scan_manager.cpp`, `src/scan_manager/parquet_split_provider.cpp`, `src/scan_manager/cached_split_provider.cpp`, `src/scan_manager/split_connector.cpp`, `src/scan_manager/split_provider.cpp`, `src/op/scan/prefetched_data_source.cpp`, `src/op/scan/cached_ranges.cpp`
- Apache Iceberg
  - Operator + tasks: `src/op/sirius_physical_iceberg_scan.cpp`, `src/op/scan/iceberg_scan_task.cpp`, `src/op/scan/iceberg_metadata_reader.cpp`
  - Manifest readers: `src/op/scan/iceberg_avro_reader.cpp`, `src/op/scan/puffin_reader.cpp`
  - Delete pipeline: `src/op/scan/iceberg_delete_pipeline.cpp`, `src/op/scan/equality_delete_filter.cpp`, `src/op/scan/positional_delete_filter.cpp`, `src/op/scan/equality_delete_mask.cu`
  - Hive partition pruning: `src/op/scan/hive_partition.cpp`
- DuckDB native scan
  - `src/op/sirius_physical_duckdb_scan.cpp`, `src/op/scan/duckdb_scan_executor.cpp`, `src/op/scan/duckdb_scan_task.cpp`, `src/op/scan/duckdb_native_metadata.cpp`, `src/op/scan/cpu_source_task.cpp`
- Pinned tables (user-curated GPU/host caches)
  - `src/pin_table.cpp` reads files via `cudf::io::chunked_parquet_reader` and inserts them into `sirius::scan_manager::sirius_scan_manager` keyed by user-provided `name`
- Local filesystem (spill / cache)
  - `src/downgrade/downgrade_executor.cpp` — Spills GPU pages to host + disk under OOM
  - `src/data/host_parquet_representation.cpp` — Host-tier columnar representation used by both spill and `pin_table(tier='host')`

**Object Storage (S3):**
- Custom AWS SigV4 v4 signer (no AWS SDK dependency) — `src/io/s3/sigv4.cpp`, `src/include/io/s3/sigv4.hpp`
  - `sigv4_signer_config` carries access key / secret key / region / service / optional session token
  - Provides `sign_request` (HTTP header injection) and `presign_url`
  - Backed by `OpenSSL::Crypto` (SHA-256 + HMAC) — `find_package(OpenSSL REQUIRED)` (`CMakeLists.txt` line 71)
- Credential providers: `src/include/io/s3/credential_provider.hpp`, `src/include/io/s3/static_credentials.hpp`, `src/include/io/s3/mock_credential_provider.hpp`, `src/io/s3/sirius_sigv4_credential_provider.cpp`
- Object-store configuration: `src/include/io/object_store_config.hpp`

**I/O Engine:**
- io_uring asynchronous reader — `src/io/uring/uring_reactor.cpp`, `src/io/uring/uring_ioctx.cpp`, `src/include/io/uring/uring_{reactor,ioctx}.hpp`
- `sirius_datasource` overrides `cudf::io::datasource` (`src/include/io/sirius_datasource.hpp`, `src/io/sirius_datasource.cpp`) — feeds GPU-native parquet decode kernels with pinned host buffers via io_uring
- Admission control + prefetch cache: `src/io/admission_control.cpp`, `src/io/prefetching_cache.cpp`, `src/io/io_context.cpp`
- URI parsing + datasource factory: `src/io/uri_parser.cpp`, `src/io/datasource_factory.cpp`
- liburing linked as `PkgConfig::LIBURING` via `pkg_check_modules(LIBURING REQUIRED IMPORTED_TARGET liburing)` (`CMakeLists.txt` line 73); `liburing` is also a vcpkg-manifest dep (`vcpkg.json`)

**Caching:**
- GPU L2/SM caches (implicit via CUDA)
- Scan result cache levels selectable per query: `none | table_gpu | table_host | parquet` (`scan_cache_level` extension option → `sirius::op::scan::cache_level`)
- Prefetch cache for parquet ranges: `src/io/prefetching_cache.cpp`
- Disk spilling: cuCascade-driven via `src/downgrade/downgrade_executor.cpp`
- Legacy host CPU cache: `src/legacy/cpu_cache.cpp` (only built when `ENABLE_LEGACY_SIRIUS=ON`)

## Authentication & Identity

**Auth Provider:**
- DuckDB has no native auth; Sirius requires `allow_unsigned_extensions=true` to load via SQL `LOAD '…'`
- For S3 access, credentials are resolved by `sirius::io::s3::sirius_sigv4_credential_provider` (`src/io/s3/sirius_sigv4_credential_provider.cpp`); static credentials supported via `static_credentials.hpp` and `mock_credential_provider.hpp` for tests
- No remote control plane / OAuth / API key authentication exists in the engine

## Monitoring & Observability

**Logging:**
- spdlog 1.8.x — global logger initialised by `InitGlobalLogger(level, dir, flush_seconds)` (`src/log/logging.hpp`)
- Configurable via SQL `SET sirius_log_level=…`, `SET sirius_log_dir=…`, `SET sirius_log_flush_seconds=…` or env vars `SIRIUS_LOG_LEVEL`, `SIRIUS_LOG_DIR`
- Compile-time defaults injected via `SIRIUS_DEFAULT_LOG_DIR`, `SIRIUS_UNITTEST_LOG_DIR`, `SIRIUS_PROJECT_ROOT` (`CMakeLists.txt` lines 485–490)
- Unit-test log directory: `build/release/extension/sirius/test/cpp/log`

**Tracing & Telemetry (Quent):**
- Rust-side exporter (`telemetry_bridge` + `rust/crates/telemetry/model`) ingests quent events and writes NDJSON when `enable_quent=true`
- C++ entry points: `src/telemetry/telemetry_context.cpp`, `src/include/telemetry/telemetry_context.hpp`
- Optional query label propagated via `gpu_execution(…, query_label='…')` or `CALL sirius_set_query_label('…')` (consumed in `SiriusExtension::GPUExecutionBind`, `src/sirius_extension.cpp` lines 418–445)

**Performance Profiling:**
- NVIDIA Nsight Systems integration via the `profiler_start` / `profiler_stop` SQL UDFs (`src/sirius_extension.cpp` lines 918–940) — wraps `cudaProfilerStart` / `cudaProfilerStop` so nsys `--capture-range=cudaProfilerApi` can scope a profile
- `tools/parse_pipeline_log.py` — Parses Sirius pipeline logs to show per-operator row counts
- Claude skills under `.claude/skills/`: `profile-analyzer/`, `optimization-advisor/`, `benchmark/`

**GPU Stream-Check (optional):**
- `option(ENABLE_STREAM_CHECK …)` (`CMakeLists.txt` line 28) — when `ON`, builds `utils/stream_check/` as a runtime-loadable library that validates absence of default-stream usage; loaded via `src/util/stream_check_wrapper.cpp`

**Crash diagnostics:**
- `sirius::util::install_segfault_backtrace_handler()` (`src/util/segfault_backtrace.hpp`, impl in `src/util/segfault_backtrace_handler.cpp`) is invoked at the top of `LoadInternal`

## CI/CD & Deployment

**Hosting:**
- Upstream: `https://github.com/sirius-db/sirius` (default branch: `dev`)
- This worktree tracks `sirius_expression_framework` branch (Phase 7b refactor — see `.planning/PROJECT.md`)

**CI Pipeline (.github/workflows/):**
- `check.yml` — pre-commit on `ubuntu-24.04`; uses `prefix-dev/setup-pixi@v0.9.5` + `pre-commit` global env
- `test.yml` — Self-hosted CUDA-13 runner (`[self-hosted, ubuntu-24.04, cuda-13, cpu-xl]`); 60-minute timeout; sccache backend auto-selects between GHA and S3 based on `SCCACHE_BUCKET`
- `distribution.yml` — Calls `sirius-db/extension-ci-tools/.github/workflows/_extension_distribution.yml@sirius` for both CUDA-12 and CUDA-13 builds on `linux_x64` and `linux_arm64` self-hosted runners; uses `build_type: ci-release`, `extra_toolchains: 'cuda;rust;protobuf'`, `vcpkg_commit: ffc071e0c08432c60c9b64f00334c0227667931b`, `duckdb_version: v1.5.1`
- Allow-listed third-party actions: `prefix-dev/setup-pixi@*`, `mozilla-actions/sccache-action@*`; all third-party actions must be pinned to a 40-char commit SHA

**Build Tooling:**
- sccache 0.15.0 — Wired into all GCC/Clang presets via `CMAKE_*_COMPILER_LAUNCHER=sccache`; `ci-release` preset switches to ccache
- mold linker — `CMAKE_LINKER_TYPE=MOLD` for non-CI presets
- `Makefile` (wrapper): targets `release`, `debug`, `relwithdebinfo`, `clang-{release,debug,relwithdebinfo}`, `legacy-release`, `ci-release`, plus `test_*` variants — each invokes `cd duckdb && cmake --preset <name> && cmake --build --preset <name>`

## Environment Configuration

**Activation env (`pixi.toml` `[activation.env]`):**
- `SCCACHE_BASEDIRS=$PIXI_PROJECT_ROOT:$CONDA_PREFIX`

**Feature env (`[feature.cuda13.activation.env]`, `[feature.cuda12.activation.env]`):**
- `CUDAARCHS` — CUDA architectures used for compilation
- `VCPKG_CUDA_VERSION` — CUDA major version forwarded to the vcpkg flow

**Runtime env vars (optional, sensible defaults):**
- `SIRIUS_LOG_LEVEL` (trace|debug|info|warn|error|critical|off)
- `SIRIUS_LOG_DIR` (default: `${CMAKE_BINARY_DIR}/log`)
- `SCCACHE_BUCKET`, `SCCACHE_GHA_ENABLED` — Switch sccache backend in CI

**Secrets:**
- None tracked in the source tree. AWS credentials for S3 reads come from the `sirius_sigv4_credential_provider` (env vars / instance metadata / static config), not from any committed file
- GitHub Actions secrets (sccache S3 bucket, distribution artifact uploads) live in `sirius-db/sirius` repo settings

**Configuration Files:**
- `pixi.toml` — Environments + features
- `vcpkg.json`, `vcpkg_ports/`, `vcpkg_triplets/` — vcpkg manifest, overlay ports, overlay triplets
- `CMakeLists.txt` — Build graph, dependency wiring, extension targets
- `extension_config.cmake` — Registers `sirius` extension with the DuckDB extension framework (`EXTENSION_VERSION dev`)
- `cmake/CMakePresets.json` — Symlinked to `duckdb/CMakePresets.json` by `scripts/pixi_activate.sh`
- `Makefile` — Preset → ninja build invocation wrapper
- `.pre-commit-config.yaml`, `.clang-format`, `.clang-tidy`, `.codespell_words`
- `rust/Cargo.toml`, `rust/crates/telemetry/bridge/CMakeLists.txt` — Rust workspace + Corrosion glue

## Webhooks & Callbacks

**Incoming:**
- None — Sirius is an in-process DuckDB extension, not a service

**Outgoing:**
- None — No outbound HTTP/notifications beyond the S3 reads issued by `sirius_datasource` for table data

## DuckDB Extension Integration Details

**Extension Loading Flow:**
1. User runs `LOAD 'path/to/sirius.duckdb_extension'` (CLI/Python) — requires `allow_unsigned_extensions=true`
2. DuckDB invokes `DUCKDB_CPP_EXTENSION_ENTRY(sirius, loader)` → `duckdb::LoadInternal` (`src/sirius_extension.cpp` line 1477)
3. `LoadInternal` installs a segfault backtrace handler, fetches `DBConfig`, registers `SiriusContextExtensionCallback`, initialises the GPU↔HOST converter registry (`sirius::converter_registry::initialize()`), populates the extension option set (`InitialGPUConfigs`), and registers the table functions (`RegisterGPUFunctions`)
4. The optimizer extension is registered with both pre- and post-optimize hooks for transparent execution

**Transparent execution hooks (`src/transparent/sirius_optimizer_extension.cpp`):**
- `sirius_pre_optimizer_hook` — Disables `IN_CLAUSE`, `COMPRESSED_MATERIALIZATION`, and `STATISTICS_PROPAGATION` optimizers for this query and stashes the original disabled-set on `SiriusContext`
- `sirius_optimizer_hook` — Restores the original disabled-set, then deep-copies the optimized `LogicalOperator` into `SiriusContext` for `OnFinalizePrepare` (`src/transparent/physical_sirius_execution.cpp`) to attempt GPU plan construction; throws ⇒ silent CPU fallback

**Explicit execution path (`gpu_execution(query)`):**
- `SiriusExtension::GPUExecutionBind` (`src/sirius_extension.cpp` line 418) constructs a `sirius::sirius_interface`, extracts + optimizes the plan, then calls `sirius::planner::sirius_physical_plan_generator::create_plan` to produce a `sirius::op::sirius_physical_operator`
- `SiriusExtension::GPUExecutionFunction` (`src/sirius_extension.cpp` line 490) calls `sirius_iface->sirius_execute_query(...)`; on error and with `enable_duckdb_fallback=true`, falls back to a CPU query via `run_internal_cpu_fallback_query`

**Memory Management Integration:**
- DuckDB allocates input batches (`duckdb::DataChunk`); Sirius copies to GPU via RMM + cuCascade reservations
- cuCascade handles spilling/reservation when GPU memory is exhausted (`src/memory/sirius_memory_reservation_manager.cpp`, `src/downgrade/downgrade_executor.cpp`)
- Results stream back to host via `src/op/result/host_table_chunk_reader.cpp` for DuckDB consumption

---

*Integration audit: 2026-05-20*
