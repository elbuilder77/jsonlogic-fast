# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`jsonlogic-fast` is a thin, production-oriented wrapper around the `jsonlogic` crate (jsonlogic-rs) that exposes the same JSON-Logic evaluation API to Rust, Python (PyO3/maturin), and JavaScript (wasm-bindgen). It does not implement JSON-Logic operators itself; operator semantics come from `jsonlogic::apply`.

## Commands

The Makefile assumes a POSIX shell with `make`, `bash`, `python3`, and `uv`. Where `make` is unavailable (e.g. plain Windows), run the underlying commands shown below directly.

```bash
make ci            # fmt + clippy + test + test-python + test-wasm (what CONTRIBUTING asks for before a PR)
make fmt           # cargo fmt --all            (CI runs: cargo fmt --all --check)
make clippy        # cargo clippy --workspace --all-targets -- -D warnings
make test          # cargo test --workspace
make test-python   # builds the extension with maturin, then runs pytest on tests/python/
make test-wasm     # cd wasm && wasm-pack test --node   (needs wasm-pack + wasm32-unknown-unknown target)
make bench-quick   # criterion smoke run (what CI runs)
make bench-python  # benchmarks/python_bench.py
```

Running a subset:

```bash
cargo test -p jsonlogic-fast <test_name_substring>     # core unit tests (in core/src/*.rs)
cargo test -p jsonlogic-fast --test compatibility      # core/tests/compatibility.rs
cargo test -p jsonlogic-fast --test proptest           # property-based tests
cd python && uv run --with maturin --with pytest bash -c "maturin develop --release && pytest ../tests/python/test_evaluate.py -k <name> -v"
```

Python tests import the compiled `jsonlogic_fast` module, so any Rust change requires re-running `maturin develop` before pytest sees it.

## Architecture

Cargo workspace with three members; the version lives in `[workspace.package]` in the root `Cargo.toml`.

- **`core/`** — crate `jsonlogic-fast` (lib name `jsonlogic_fast`). Essentially all logic is in `core/src/lib.rs`; `error.rs` defines `RuleEngineError`, `extract.rs` defines `extract_f64` numeric coercion.
- **`python/`** — crate `jsonlogic-fast-python`, a cdylib built by maturin into the Python module `jsonlogic_fast` (`module-name` in `python/pyproject.toml`). Tests live at the repo root in `tests/python/`, not under `python/`.
- **`wasm/`** — crate `jsonlogic-fast-wasm`, published to npm via `wasm-pack` (output in `wasm/pkg/`, which `examples/browser_wasm_demo` depends on via `file:`).

### Bindings mirror core one-to-one

Both binding crates import every core function under a `core_*` alias and wrap it with no extra logic. Adding or changing a public API therefore means touching all three: the core function, a `#[pyfunction]` **plus** its registration in the `#[pymodule]` function at the bottom of `python/src/lib.rs`, and a `#[wasm_bindgen]` function (free functions carry a `_wasm` suffix; `CompiledRule` methods do not). Update tests in `core/`, `tests/python/`, and `wasm/tests/` accordingly.

Binding-specific shapes to preserve:
- **Python**: `evaluate*` return native Python objects via `pythonize`; `*_json` variants return JSON strings; `get_core_info` returns a JSON string; `evaluate_batch_numeric_detailed` returns a `(values, errors)` tuple with `""` meaning no error. All errors surface as `ValueError`.
- **WASM**: every result is returned as a JSON string (except `evaluate_numeric_wasm`/`validate_rule_wasm`). Batch functions take contexts as a single JSON **array** string, which is parsed and re-serialized per element before calling core. Errors are thrown as plain string `JsValue`s.

### Evaluation semantics (core)

- **`CompiledRule`** just holds the pre-parsed `serde_json::Value` rule; "compiled" means "parsed once", not a separate IR.
- **Batch families** have deliberately different failure behavior:
  - `evaluate_batch` — lossy: a bad context yields `null` at that index; only an invalid rule errors.
  - `evaluate_batch_detailed` / `evaluate_batch_numeric_detailed` — per-item `{result, error}` (numeric uses `0.0` on error).
  - `evaluate_batch_numeric` — errors if any item fails.
  - `*_strict` — fail-fast, and intentionally **sequential** (plain loop with `?`).
- **Parallelism**: `map_contexts` and `available_threads` have two `cfg(target_arch = "wasm32")` variants — Rayon `par_iter` on native, sequential iterator on WASM. `rayon` is a non-wasm target dependency only, so any Rayon use in core must stay behind that cfg.
- **Numeric coercion** (`extract_f64`): number → f64, numeric string → parsed, bool → 1/0, null → 0, arrays/objects → error. Coercion error messages must not echo the offending value (enforced by `test_extract_f64_exposure_protection`).
- **`validate_rule`** is not a static check: it evaluates the rule against a fixed sample context (`default_validation_context`) and reports whether evaluation succeeds.
- Errors are stringly typed across the FFI boundary: bindings use the `Display` text of `RuleEngineError` (e.g. `"Error parsing context: ..."`, `"Numeric coercion error: ..."`), and tests in all three layers assert on these substrings — changing an error message is a cross-layer change.

## Versioning and release (enforced by CI)

Any PR touching `core/`, `python/`, or `wasm/` must bump the version above what is on crates.io, or the `check-version` job (`scripts/check_version.py`) fails. A bump must be applied in four places kept in sync by hand:
1. `[workspace.package] version` in root `Cargo.toml`
2. the `jsonlogic-fast = { version = ..., path = "../core" }` dependency in `python/Cargo.toml`
3. the same dependency in `wasm/Cargo.toml`
4. `version` in `python/pyproject.toml`

Also add a `CHANGELOG.md` entry. Releasing: push a `vX.Y.Z` tag matching `Cargo.toml` → `publish-crates.yml` (runs in the `crates-io` environment) tests core, publishes to crates.io via Trusted Publishing (OIDC, no stored token), and creates a GitHub Release. That release is created with `GITHUB_TOKEN`, so it does **not** trigger `publish-pypi.yml`; PyPI publishing is a manual `workflow_dispatch`. The CI `package` job runs `cargo publish --dry-run -p jsonlogic-fast` on every PR.

## Other notes

- MSRV of the published crate is Rust 1.80 (`rust-version` in the workspace, driven by `rayon` 1.12), enforced by the `msrv` CI job. The Python crate overrides it to 1.83 (required by `pyo3` 0.29) and builds against `abi3-py39`.
- CI also runs `cargo-deny` (`deny.toml` license allowlist, unknown registries/git sources denied), `rustsec/audit-check`, and `cargo doc --no-deps` in `core/`.
- The Rust files in the root `examples/` directory are **not** registered as Cargo targets (the root manifest is a virtual workspace and `core/` has no `examples/` dir), so `cargo run --example` will not find them. The Python examples require the extension to be built first.
- `COMPATIBILITY.md` documents operator coverage and known divergences from the JS reference implementation; update it when behavior changes.
