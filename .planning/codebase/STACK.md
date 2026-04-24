# Technology Stack

**Analysis Date:** 2026-05-20

## Languages

**Primary:**
- C++ (C++20 standard) — Core GPU engine, planner, expression executor, operators, DuckDB extension entry points
- CUDA (CUDA 12.x or 13.x; CUDA standard 20) — Device kernels and NVIDIA library wrappers; CUDA arch list driven by pixi feature
- Rust (edition 2024) — `telemetry-bridge` static lib that exposes quent NDJSON telemetry to C++ via CXX (`rust/crates/telemetry/bridge/`, `rust/crates/telemetry/model/`)

**Secondary:**
- Python 3.10+ — Optional `duckdb-python` bindings via pybind11 / scikit-build-core (`duckdb-python/` submodule)
- Shell/Bash — Environment helpers: `scripts/pixi_activate.sh`, `scripts/generate_nightly_env.sh`, `scripts/vcpkg.sh`, `scripts/extension-upload.sh`, `scripts/run-tpch.sh`

## Runtime

**Environment:**
- Pixi (>= 0.59) — Primary dev/CI environment; pixi-version v0.65.0 pinned in CI (`.github/workflows/test.yml`, `.github/workflows/check.yml`)
- CUDA runtime 12.x or 13.x — Chosen per pixi environment (`default` = `cuda13`; `cuda12` environment for older drivers)
- NVIDIA GPU compute architectures:
  - cuda13 (default): `75-real;80-real;86-real;90a-real;100f-real;120a-real;120` (Turing → Blackwell, sm_75–sm_120)
  - cuda12: `75-real;80-real;86-real;90a-real` (Turing → Hopper only)
- Linux only (`pixi.toml` platforms: `linux-64`, `linux-aarch64`)

**Package Manager:**
- Pixi for conda-channel dependencies (channels: `rapidsai`, `conda-forge`)
- vcpkg for the alternate static-link build (manifest at `vcpkg.json`, baseline `ffc071e0c08432c60c9b64f00334c0227667931b`, registered as `vcpkg/` git submodule)
- Cargo workspace for Rust crates (`rust/Cargo.toml`, resolver 3, members `crates/telemetry/bridge` and `crates/telemetry/model`)

## Frameworks

**Build System:**
- CMake — `cmake_minimum_required(VERSION 3.30.4 FATAL_ERROR)` in `CMakeLists.txt`; presets require CMake 3.26+
- Ninja generator (selected via `cmake/CMakePresets.json`)
- DuckDB extension Makefile shim — `Makefile` includes targets that drive `extension-ci-tools/makefiles/duckdb_extension.Makefile` indirectly (the wrapper now `cd $(DUCKDB_DIR) && cmake --preset …`)
- Corrosion (FetchContent v0.6.1) — imports the Rust `telemetry-bridge` static lib into the CMake graph (`rust/crates/telemetry/bridge/CMakeLists.txt`)
- CXX (Rust crate `cxx = "1"` + build crate `cxx-build = "1"`) — Generates the C++/Rust FFI headers consumed by `target_link_libraries(... telemetry_bridge)`
- sccache 0.15.0 — Compiler cache wired into all GCC/Clang presets via `CMAKE_*_COMPILER_LAUNCHER=sccache`; CI presets switch to `ccache` (Docker vcpkg flow)
- mold linker — Default `CMAKE_LINKER_TYPE=MOLD` for all non-CI presets

**GPU/CUDA Libraries:**
- RAPIDS cuDF 26.04.x — `find_package(cudf REQUIRED CONFIG)`; primary GPU dataframe API for joins, aggregates, sorts, scans, expressions
- RMM (RAPIDS Memory Manager) — Pulled transitively with cudf; explicit dependency for the duckdb-python build (`librmm = "*"` in `[feature.dev-libs.dependencies]`)
- cuCascade — Vendored git submodule `cucascade/`; built as `cucascade_static` and linked as `cuCascade::cucascade`; tiered memory + reservation + GPU↔HOST converters

**Testing:**
- Catch2 — Headers consumed from `duckdb/third_party/catch` (included via `target_include_directories(sirius_unittest BEFORE …)`)
- DuckDB SQL Logic Tests — Tests under `test/sql/` (`tpch-sirius.test`, `clickbench-sirius.test`, …) executed by `build/release/test/unittest`
- Pixi + `make test` Makefile target — Wraps the standard DuckDB unittest runner

**Development/CI:**
- pre-commit (`.pre-commit-config.yaml`): standard hooks v6.0.0, black v25.1.0, fix-smartquotes v0.7.1, mirrors-clang-format v20.1.4, codespell v2.4.1, cmake-format / cmake-lint v0.6.13
- GitHub Actions: `.github/workflows/check.yml` (pre-commit), `.github/workflows/test.yml` (CUDA 13 self-hosted unit tests), `.github/workflows/distribution.yml` (CUDA 12 & 13 distribution artifacts)
- `prefix-dev/setup-pixi@v0.9.5` and `mozilla-actions/sccache-action` are the only allow-listed non-GitHub actions

## Key Dependencies

**Critical (linked into both `sirius_extension` and `sirius_loadable_extension` per `CMakeLists.txt` lines 332–349):**
- `cudf::cudf` — GPU dataframe library (libcudf 26.04.x)
- `rmm::rmm` — Pulled in by cudf, used directly for device buffers and streams
- `spdlog::spdlog` 1.8.x — Structured logging
- `cuCascade::cucascade` — Tiered memory + scan_manager backend (submodule `cucascade/`)
- `yaml-cpp::yaml-cpp` — YAML config loader (`src/config.cpp`, `src/sirius_config.cpp`)
- `telemetry_bridge` — Rust static lib exposing quent telemetry via CXX (`rust/crates/telemetry/bridge/`)

**Static-extension-only extras (`sirius_extension` target):**
- `PkgConfig::NUMA` — libnuma host allocations / NUMA affinity
- `PkgConfig::LIBURING` — io_uring asynchronous file I/O backend (`src/io/uring/`)
- `OpenSSL::Crypto` — SHA-256 + HMAC for AWS SigV4 signing (`src/io/s3/sigv4.cpp`)
- `absl::any_invocable` — Type-erased callable for the task scheduler

**Loadable-extension extras (`sirius_loadable_extension` target):**
- `PkgConfig::LIBURING`, `OpenSSL::Crypto` (linked statically into the loadable artifact)

**DuckDB integration:**
- `duckdb` = 1.5.2 (pixi `[feature.dev-libs.dependencies]`); `duckdb/` git submodule pinned at tag `v1.5.2` (`8a5851971fae891f292c2714d86046ee018e9737`)
- `extension-ci-tools` git submodule on `sirius-db/extension-ci-tools` `sirius` branch
- `extension_config.cmake` registers this repo as `sirius` with `EXTENSION_VERSION dev`
- `substrait/` git submodule pinned at `v1.2.0-17-g703de07` (currently inactive — header include in `src/sirius_extension.cpp` is commented out)
- `.github/workflows/distribution.yml` builds against `duckdb_version: v1.5.1` (CI build pin) while the dev environment tracks `v1.5.2`

**Conda-only dev libs (`[feature.dev-libs.dependencies]`):**
- `libabseil`, `libcurand-dev`, `sqlite = 3.*`, `liburing`, `rust-src`

**vcpkg manifest (`vcpkg.json`):**
- Top-level: `cudf`, `yaml-cpp`, `abseil`, `numactl`, `liburing`
- Overrides: `dlpack` 0.8, `flatbuffers` 24.3.25
- Overlay ports: `vcpkg_ports/`; overlay triplets: `vcpkg_triplets/`

**Rust workspace (`rust/Cargo.toml`):**
- `cxx = "1"` + `cxx-build = "1"` for FFI bridges
- `instrumentation-model` (local crate, depends on `quent-attributes`, `quent-model`, `quent-query-engine-model`, `quent-stdlib` — all pinned to `rapidsai/quent` rev `871f8f45fd35288945a13636df8cadf50f7dbcc2`)
- `quent-codegen` (same quent rev, build-time only)

**Build/Development:**
- `cmake = "4.*"`, `cuda-nvcc = "*"`, `cuda-nvml-dev = "*"`, `cxx-compiler = "*"`, `make = "*"`, `ninja = "*"`
- `clang = "21.*"`, `clang-tools = "21.*"`, `pkg-config = "*"`, `mold = "*"`, `libprotobuf = "*"`, `rust = "*"`, `bc = "*"`, `wget = "*"`, `unzip = "*"`
- `pre-commit = "*"`, `identify >= 2.6`

## Configuration

**Environment Setup:**
- `pixi.toml` — Workspace + activation scripts + per-feature CUDA settings
- Environments: `default` (`dev-libs + cuda13`), `cuda12` (`dev-libs + cuda12`), `duckdb-python` (Python bindings), `vcpkg` (vcpkg + cuda13), `vcpkg-cuda12`, `nightly-runner` (lightweight env for running nightly cudf builds)
- `scripts/pixi_activate.sh` — Symlinks `clang-cpp → clang++`, links `cmake/CMakePresets.json` into `duckdb/CMakePresets.json`, writes `build/sirius_pixi_env_for_clion.sh`
- `scripts/generate_nightly_env.sh`, `envs/nightly/pixi.toml` — Pulls cuDF nightly builds for canary testing

**Build Presets (`cmake/CMakePresets.json`, symlinked to `duckdb/CMakePresets.json`):**
- GCC: `debug`, `release`, `relwithdebinfo`
- Clang: `clang-debug`, `clang-release`, `clang-relwithdebinfo` (uses `$CONDA_PREFIX/bin/clang++` + `clang` host compiler)
- vcpkg static-link: `vcpkg-debug`, `vcpkg-release`, `vcpkg-relwithdebinfo`
- `legacy-release` — toggles `ENABLE_LEGACY_SIRIUS=ON` (builds `src/legacy/` content)
- `ci-release` — Docker vcpkg + ccache (used by `.github/workflows/distribution.yml`)

**Code Quality:**
- `.clang-format` — C++/CUDA formatting rules (enforced by pre-commit + `make`-level CI)
- `.clang-tidy` — Static analysis ruleset (used in IDE flow; `WarningsAsErrors: '*'` per project rules)
- `.codespell_words` — Custom spell-check allowlist (`aktion`, `ans`, `foto`, `fpr`, `ist`, `thirdparty`)
- `.pre-commit-config.yaml` — Top-level hook registry; excludes `*.csv` and `legacy/queries/*`
- Optional CMake feature flag: `option(ENABLE_STREAM_CHECK "Enable stream check library …" OFF)` — builds `utils/stream_check/` for default-stream usage validation

**Build Artifacts:**
- Static extension: `build/release/extension/sirius/sirius.duckdb_extension`
- Loadable extension: `build/release/extension/sirius/sirius_loadable.duckdb_extension`
- Unit tests: `build/release/extension/sirius/test/cpp/sirius_unittest`
- Test logs: `build/release/extension/sirius/test/cpp/log` (Catch2 logger)
- Parquet benchmark binary: `build/release/extension/sirius/test/io/parquet_benchmark`
- DuckDB CLI: `build/release/duckdb` (built by the `duckdb` target in `MAIN_BUILD_TARGETS`)

## Platform Requirements

**Development:**
- Linux x86_64 or aarch64
- NVIDIA GPU with Compute Capability ≥ 7.5 (sm_75 minimum)
- 8+ GB GPU memory recommended for TPC-H SF1 (parquet caching + reservation)
- pixi >= 0.59 (CI pins v0.65.0)

**Build Dependencies:**
- GCC ≥ 13 or Clang 21 (C++20)
- CUDA 12.x or 13.x toolkit (host compiler must be supported by chosen `nvcc`)
- Rust toolchain (`rust` + `rust-src` from conda; Corrosion fetches its own copy on demand if missing)
- Python 3.10+ with pip, pybind11, scikit-build-core for the `duckdb-python` environment
- Self-hosted CI runners with CUDA 13 GPU (`runs-on: [self-hosted, ubuntu-24.04, cuda-13, cpu-xl]`)

**Production/Deployment:**
- Loaded into DuckDB as either `.duckdb_extension` (static-linked) or `.duckdb_loadable_extension` (dynamic)
- Distribution workflow produces both CUDA 12 and CUDA 13 artifacts for `linux_amd64` and `linux_arm64`
- Requires `allow_unsigned_extensions=true` to load via SQL `LOAD '…'` until artifacts are signed

## Git Submodules

| Path | Repo | Pinned commit / branch |
|------|------|------------------------|
| `duckdb` | duckdb/duckdb | `v1.5.2` |
| `extension-ci-tools` | sirius-db/extension-ci-tools | `sirius` branch (`245e2c29…`) |
| `substrait` | sirius-db/duckdb-substrait-extension | `v1.2.0-17-g703de07` (currently unused) |
| `cucascade` | NVIDIA/cuCascade | `main-9-g9ceebaa` |
| `duckdb-python` | duckdb/duckdb-python | `v1.4.1-511-g2aea44e` |
| `.claude/claude-tools` | sirius-db/claude-tools | `main` |
| `vcpkg` | Microsoft/vcpkg | `2026.01.16-492-gffc071e0c0` |

---

*Stack analysis: 2026-05-20*
