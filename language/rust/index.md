# Rust Language Guidelines

This directory contains Rust-specific coding guidelines for **Rust 2024 edition** (stable since Rust 1.85).

## Available Chunks

- **basics.md** - Toolchain and edition, project layout, ownership, iterators, pattern matching, naming
- **error-handling.md** - `Result`, error types, `thiserror` vs `anyhow`, panic policy
- **types-traits.md** - Newtypes, builders, typestate, traits vs generics vs `dyn`, `async fn` in traits
- **async.md** - tokio, blocking work, cancellation safety, channels, shutdown
- **testing.md** - Unit, integration and doc tests, fakes, property tests, snapshots
- **tooling.md** - `Cargo.toml` lints, rustfmt, clippy, cargo-deny, CI, supply chain
- **unsafe-ffi.md** - `unsafe` discipline, safety comments, Miri, C interop
- **performance.md** - Measure first, allocation, iterators, profiles, benchmarks
- **web-cli.md** - axum services, serde, clap, tracing, configuration, graceful shutdown

## Conventions Used In These Files

- Code blocks tagged `rust` compile on their own with the standard library only.
- Blocks tagged `rust,ignore` are fragments or need third-party crates.
- Crate names are recommendations, not requirements. Check the crate's current release before copying a version.
