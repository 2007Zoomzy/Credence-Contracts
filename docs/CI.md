# CI Linting Guide for Contract Contributors

> Maintainer-expected linter commands before a contract PR is reviewed.

This document lists the **exact commands** maintainers expect you to run before
opening a PR that touches contracts. Running them locally prevents CI failures
and keeps reviews focused on logic, not style.

## The CI Gate

The required CI workflow is `.github/workflows/contracts-ci.yml`. It runs five
jobs — `fmt`, `test`, `clippy`, `wasm`, and `no-bare-panic` — against the four
**gated packages** listed in `scripts/ci-gated-packages.txt`:

```
credence_arbitration
timelock
dispute_resolution
testutils
```

**To add a crate to the gate:** add its package name (one line) to
`scripts/ci-gated-packages.txt` and ensure it passes all five jobs locally
before opening the PR. No other file needs to change.

---

## Quick Reference — Run All Lints

```bash
# 1. Formatting
cargo fmt --all -- --check

# 2. Standard clippy (gated packages only)
cargo clippy -p credence_arbitration --all-targets --locked -- -D warnings
cargo clippy -p timelock --all-targets --locked -- -D warnings
cargo clippy -p dispute_resolution --all-targets --locked -- -D warnings
cargo clippy -p testutils --all-targets --locked -- -D warnings

# 3. Security-focused clippy
cargo clippy --all-targets -- \
  -W clippy::arithmetic_side_effects \
  -W clippy::unwrap_used \
  -W clippy::expect_used \
  -W clippy::panic \
  -W clippy::todo \
  -W clippy::unimplemented \
  -W clippy::indexing_slicing \
  -W clippy::cast_possible_truncation \
  -W clippy::cast_sign_loss \
  -D warnings

# 4. Tests (gated packages only)
cargo test -p credence_arbitration --locked
cargo test -p timelock --locked
cargo test -p dispute_resolution --locked
cargo test -p testutils --locked

# 5. Coverage (per-crate, ≥ 95%)
cargo llvm-cov --package credence_bond --fail-under-lines 95
cargo llvm-cov --package credence_delegation --fail-under-lines 95
cargo llvm-cov --package timelock --fail-under-lines 95

# 6. Error code wire stability
cargo test -p credence_errors error_codes_wire

# 7. Dependency audit
cargo audit

# 8. Unsafe code detection
cargo geiger
```

---

## 1. Formatting

**Command:**
```bash
cargo fmt --all -- --check
```

**What it checks:** Consistent code formatting across the entire workspace.

**Auto-fix:**
```bash
cargo fmt --all
```

**CI job:** `contracts-ci.yml` (`fmt` job)

---

## 2. Standard Clippy Lints

**Command:**
```bash
cargo clippy -p credence_arbitration --all-targets --locked -- -D warnings
# (repeat for each package in scripts/ci-gated-packages.txt)
```

**What it checks:** The full Rust lint suite (correctness, style, complexity,
perf) on every crate and every target (lib, tests, benches, examples).

**Tips:**
- Fix warnings in code rather than suppressing them with `#[allow]`.
- Prefer `cargo clippy --fix` for auto-fixable lints when safe.

**CI job:** `contracts-ci.yml` (`clippy` job)

---

## 3. Security-focused Clippy Lints

**Command:**
```bash
cargo clippy --all-targets -- \
  -W clippy::integer_arithmetic \
  -W clippy::unwrap_used \
  -W clippy::expect_used \
  -W clippy::panic \
  -W clippy::todo \
  -W clippy::unimplemented \
  -W clippy::indexing_slicing \
  -W clippy::cast_possible_truncation \
  -W clippy::cast_sign_loss \
  -D warnings
```

**What it checks:** Patterns that are warnings in the Rust ecosystem but are
blocking failures for the Credence security pipeline:

| Lint | Risk | Approved alternative |
|---|---|---|
| `unwrap_used` / `expect_used` | Silent panics in production | `unwrap_or_else(\|\| panic_with_error!(…))` |
| `panic` / `todo` / `unimplemented` | Unexpected halting | `panic_with_error!` with wire-stable codes |
| `arithmetic_side_effects` | Unchecked overflow/underflow | `checked_add`, `saturating_add` |
| `indexing_slicing` | Out-of-bounds panic | `.get()` with explicit error handling |
| `cast_possible_truncation` | Silent precision loss | Explicit bounds checks before casting |
| `cast_sign_loss` | Unexpected sign change | `try_from` with error handling |

**CI job:** `security.yml`

---

## 4. Tests

**Command:**
```bash
cargo test -p credence_arbitration --locked
# (repeat for each package in scripts/ci-gated-packages.txt)
```

**What it checks:** All unit tests, integration tests, and doc-tests for the
gated packages.

**Tips:**
- Use `cargo test -p <crate> <test_name>` for targeted runs during development.

**CI job:** `contracts-ci.yml` (`test` job)

---

## 5. Coverage Gate (≥ 95%)

**One-time setup:**
```bash
rustup component add llvm-tools-preview
cargo install cargo-llvm-cov --locked
```

**Commands:**
```bash
cargo llvm-cov --package credence_bond --fail-under-lines 95
cargo llvm-cov --package credence_delegation --fail-under-lines 95
cargo llvm-cov --package timelock --fail-under-lines 95
```

**What it checks:** Per-crate line coverage meets or exceeds 95%.

**Tip:** Generate an HTML report to find uncovered lines:
```bash
cargo llvm-cov --package credence_bond --open
```

**CI job:** `coverage.yml` (out of scope for the main gate; thresholds TBD)

---

## 6. Error Code Wire Stability

**Command:**
```bash
cargo test -p credence_errors error_codes_wire
```

**What it checks:** `ContractError` variant discriminants match their
documented wire-stable codes. Variants must not be renumbered or deleted
after deployment.

**Policy:** See `docs/error-codes-wire.md` for the bump procedure when
adding new errors.

---

## 7. Dependency Vulnerability Scan

**One-time setup:**
```bash
cargo install cargo-audit --version 0.22.0 --locked
```

**Command:**
```bash
cargo audit
```

**What it checks:** Known vulnerabilities in the dependency tree
(advisory database from the RustSec project).

**CI job:** `security.yml`

---

## 8. Unsafe Code Detection

**One-time setup:**
```bash
cargo install cargo-geiger --version 0.12.0 --locked
```

**Command:**
```bash
cargo geiger
```

**What it checks:** All usage of `unsafe` blocks in the dependency tree.
The Credence project has a zero-unsafe policy.

**CI job:** `security.yml`

---

## Common Issues and Fixes

### "error: unused import"
Run `cargo fix --allow-dirty` or remove the import manually.

### Coverage below 95%
Generate the HTML report (`cargo llvm-cov --open`) and add tests for the
uncovered lines. Pay special attention to error paths (panics, guard
branches) which are often uncovered.

### "Error(Contract, #N)" in test output
This is expected for tests that exercise panic paths. Use
`#[should_panic(expected = "Error(Contract, #N)")]` to assert the
correct error code.

### Cargo.lock out of date
Run `cargo update` and commit the updated lockfile.

---

## Resources

- [`CONTRIBUTING.md`](../CONTRIBUTING.md) — full development workflow
- [`docs/testing.md`](testing.md) — test and coverage setup
- [`docs/error-codes-wire.md`](error-codes-wire.md) — error code wire-stability
- [`docs/errors.md`](errors.md) — canonical error code listing
- [`docs/no-dynamic-strings.md`](no-dynamic-strings.md) — dynamic string prohibition
