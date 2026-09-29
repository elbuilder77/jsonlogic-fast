# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

## [0.2.0] - 2026-09-29

This release contains breaking changes to the Rust API compared to `0.1.7`.

### Removed (breaking)
- Removed the public helpers `extract::extract_bool`, `extract::extract_string`, `extract::extract_array`, and `extract::extract_object`. Use `serde_json::Value` accessors (`as_bool`, `as_str`, `as_array`, `as_object`) on the evaluation result instead.
- Removed the public `serialize` and `serialize_value` functions. Use `serde_json::to_string` directly.

### Added
- **`CompiledRule` API:** A new native object that parses JSON rules once and allows high-performance repeated evaluations. Exposed in Rust, Python, and WASM.
- **Strict Batch Variants:** Added `evaluate_batch_strict` and `evaluate_batch_numeric_strict` to fail fast on errors, complementing the default fault-tolerant batch methods.
- **Compliance Suite:** Integrated full JSONLogic specification compliance tests and Property-Based Testing (fuzzing) using `proptest`.
- **Examples:** Added `fraud_scoring.rs`, `feature_flags.rs`, `credit_eligibility.py`, and a full Vite-based `browser_wasm_demo/`.
- **Documentation:** Added `BENCHMARKS.md`, `COMPATIBILITY.md`, `SECURITY.md`, and completely rewrote `README.md` for production readiness.

### Changed
- **MSRV raised to Rust 1.80.** The previously declared `1.75` was inaccurate, since `rayon` 1.12 requires 1.80. The MSRV is now verified in CI. The Python bindings crate requires Rust 1.83 (PyO3 0.29).
- **Numeric coercion errors no longer echo the offending value.** `extract_f64` messages now report only the value type (e.g. `Expected numeric result, got: object`), so evaluation results are not leaked into error messages or logs.
- **Dependency Upgrades:**
  - Upgraded `rayon` from `1.11` to `1.12`.
  - Upgraded `thiserror` from `1.0` to `2.0` (internal only; not part of the public API).
  - Upgraded `criterion` from `0.5` to `0.8`.
  - Python bindings: upgraded `pyo3` and `pythonize` to `0.29`.

### Security
- Updated `anyhow` to `1.0.103` (RUSTSEC-2026-0190) and `crossbeam-epoch` to `0.9.20` in the lockfile.

### CI / Release
- crates.io releases are published from `v*.*.*` tags using Trusted Publishing (OIDC), with no stored API token.
- New CI jobs: MSRV check (`cargo check` on Rust 1.80) and a `cargo publish --dry-run` packaging check.
- Added a version-bump guardrail (`scripts/check_version.py`) for PRs touching `core/`, `python/`, or `wasm/`.
