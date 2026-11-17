# Rust Tooling, Lints And CI

## The Required Gate

Every change must pass these locally and in CI, in this order (cheapest first):

```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace --all-features
cargo test --workspace --doc          # needed when you use nextest
cargo doc --workspace --no-deps       # with RUSTDOCFLAGS="-D warnings" in CI
cargo deny check                      # licenses, advisories, bans, sources
```

## Cargo.toml: Lints, Profiles, Workspace

Configure lints in `Cargo.toml` so editors, CI and teammates all see the same rules. In a workspace, define them once.

```toml
[workspace]
members = ["crates/*"]
resolver = "3"            # MSRV-aware resolver; required explicitly in virtual workspaces

[workspace.package]
edition = "2024"
rust-version = "1.88"
license = "MIT OR Apache-2.0"

[workspace.lints.rust]
unsafe_code = "forbid"                  # relax per crate only where unsafe is justified
missing_debug_implementations = "warn"
unreachable_pub = "warn"

[workspace.lints.clippy]
all = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }
unwrap_used = "warn"
expect_used = "warn"
panic = "warn"
dbg_macro = "warn"
todo = "warn"
# Pedantic lints that are usually noise:
module_name_repetitions = "allow"
missing_errors_doc = "allow"
must_use_candidate = "allow"

[workspace.lints.cargo]
unused_dependencies = "warn"            # Cargo 1.100+

[profile.release]
lto = "thin"
codegen-units = 1
panic = "abort"                          # smaller and faster; only if you do not rely on unwinding
strip = "symbols"

[profile.dev.package."*"]
opt-level = 1                            # faster dependencies in debug builds
```

```toml
# crates/core/Cargo.toml
[package]
name = "core-lib"
version = "0.1.0"
edition.workspace = true
rust-version.workspace = true
license.workspace = true

[lints]
workspace = true
```

- Lint names use underscores (`unexpected_cfgs`), not hyphens; Cargo warns on hyphenated names since 1.99.
- Use `priority = -1` for lint groups so individual lints override them.
- Workspace members on edition 2024 can set `default-features = false` on an inherited dependency (Cargo 1.99+). On older editions it is ignored with a warning.
- Allow `unwrap_used`/`expect_used`/`panic` in tests via `clippy.toml`: `allow-unwrap-in-tests = true`, `allow-expect-in-tests = true`, `allow-panic-in-tests = true`.

## Features

- Features must be **additive**: enabling a feature must never remove an API or change behaviour incompatibly, because Cargo unifies features across the dependency graph.
- Keep `default` features small. Make heavy or optional dependencies opt-in: `serde = { version = "1", optional = true }`.
- Test the matrix that matters: `--no-default-features`, `--all-features`, and your documented combinations (`cargo hack --feature-powerset` for libraries).

## rustfmt

```toml
# rustfmt.toml  (stable options only)
edition = "2024"
max_width = 100
use_field_init_shorthand = true
```

- Do not hand-format. Run `cargo fmt` on save and check it in CI.
- Options such as `imports_granularity` and `group_imports` are nightly-only; do not depend on them in a stable-only project.

## Clippy

- Treat clippy warnings as errors in CI (`-D warnings`) but keep them as warnings locally.
- Deliberate exceptions use `#[expect(clippy::lint, reason = "...")]` on the smallest item possible. Do not add crate-wide `#![allow(clippy::all)]`.
- Useful non-default lints to consider: `clippy::await_holding_lock`, `clippy::large_futures`, `clippy::cast_possible_truncation`, `clippy::indexing_slicing` (strict code), `clippy::undocumented_unsafe_blocks`.

```toml
# clippy.toml
msrv = "1.88"
allow-unwrap-in-tests = true
allow-expect-in-tests = true
allow-panic-in-tests = true
```

## Dependencies And Supply Chain

| Tool | Purpose |
|------|---------|
| `cargo deny check` | Licenses, RustSec advisories, duplicate versions, allowed registries and git sources |
| `cargo audit` | RustSec advisory check against `Cargo.lock` |
| `cargo machete` / `cargo +nightly udeps` / `[lints.cargo] unused_dependencies` | Find unused dependencies |
| `cargo vet` or `cargo crev` | Record who has audited which crate versions (high-assurance projects) |
| `cargo semver-checks` | Catch accidental breaking changes in library releases |
| `cargo update --dry-run`, `cargo outdated` | See available upgrades |

```toml
# deny.toml (start from `cargo deny init`, then tighten)
[advisories]
yanked = "deny"

[licenses]
allow = ["MIT", "Apache-2.0", "Unicode-3.0"]

[bans]
multiple-versions = "warn"
wildcards = "deny"

[sources]
unknown-registry = "deny"
unknown-git = "deny"
```

- Commit `Cargo.lock` for binaries **and** libraries; CI builds with `--locked`.
- Add dependencies deliberately. Check maintenance status, download count, `unsafe` use (`cargo geiger`), build scripts and proc-macros. Every dependency is code you ship.
- Prefer a small number of well-known crates over many tiny ones. Pin by minimum compatible version (`serde = "1"`), not `*`.
- **Dependency cooldown.** Cargo 1.100 can ignore versions published only recently, which blunts typosquats and hijacked releases that are usually yanked within days:

```toml
# .cargo/config.toml
[registry]
global-min-publish-age = "7 days"

[resolver]
incompatible-publish-age = "deny"    # versions already in Cargo.lock are still allowed
```

- Review `build.rs` and procedural macros of new dependencies; they run arbitrary code at build time.
- Never run `cargo install` for a tool you have not vetted. Pin tool versions in CI (`--locked --version`).

## Toolchain Pinning And MSRV

- `rust-toolchain.toml` pins the toolchain for the repository so everyone formats and lints the same way.
- Libraries declare `rust-version` and CI tests that exact version (`cargo +1.88 check --locked`). Raise MSRV deliberately and note it in the changelog.
- With `resolver = "3"`, Cargo prefers dependency versions compatible with your `rust-version`.

## CI Example (GitHub Actions)

Pin third-party actions to a full commit SHA in real pipelines (shown here as tags for readability).

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

env:
  CARGO_TERM_COLOR: always
  RUSTFLAGS: -D warnings
  RUSTDOCFLAGS: -D warnings

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --all --check
      - run: cargo clippy --workspace --all-targets --all-features --locked
      - run: cargo test --workspace --all-features --locked
      - run: cargo doc --workspace --no-deps --locked

  deny:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: EmbarkStudios/cargo-deny-action@v2

  msrv:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: "1.88"
      - run: cargo check --workspace --locked
```

- Since Cargo 1.99, incremental compilation is off by default when `CI` is set. Do not re-enable it; rely on `rust-cache`.
- Cache `~/.cargo` and `target/` keyed on `Cargo.lock`. Do not cache `target/` for release builds you ship.
- Run `cargo +nightly miri test` on a schedule for crates with `unsafe` (see `unsafe-ffi.md`).

## Release Hygiene

- Libraries: `cargo semver-checks`, changelog, `cargo publish --dry-run`, tag from CI. Use trusted publishing with short-lived tokens where supported.
- Binaries: build with `--locked --release`, produce checksums, and consider `cargo-dist` for cross-platform artifacts. Embed version info with `env!("CARGO_PKG_VERSION")`.
- Docker: multi-stage build, `cargo-chef` for dependency layer caching, run as non-root on a minimal base (`distroless` or `debian-slim`).

## Quick Checklist

- [ ] Lints live in `[lints]`/`[workspace.lints]`, not scattered `#![allow]`s
- [ ] CI runs fmt, clippy `-D warnings`, tests, doc tests, docs and `cargo deny`
- [ ] `Cargo.lock` is committed and CI uses `--locked`
- [ ] `rust-toolchain.toml` and `rust-version` are set and tested
- [ ] New dependencies are reviewed (maintenance, licence, unsafe, build scripts)
