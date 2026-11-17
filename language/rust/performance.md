# Rust Performance

## Measure First

Rust is fast by default; most slowness is algorithmic, I/O-bound or from accidental allocation. Do not optimise without a measurement.

1. Write the clear version first. Make it correct and tested.
2. Benchmark in `--release` with realistic data. Debug builds are 10-100x slower and mislead.
3. Profile to find where the time goes, change one thing, measure again.
4. Keep the benchmark in the repo so regressions are visible.

| Tool | Use |
|------|-----|
| `criterion` or `divan` | Statistical micro-benchmarks (`benches/`) |
| `cargo flamegraph`, `samply`, `perf` | CPU profiles and flamegraphs |
| `cargo bloat`, `cargo llvm-lines` | Binary size and monomorphisation cost |
| `dhat`, `heaptrack`, `tracking-allocator` | Allocation counts and peaks |
| `cargo asm` / `cargo-show-asm` | Inspect generated assembly |
| `tokio-console` | Async task stalls and wakeups |
| `hyperfine` | End-to-end CLI timing |

```toml
# Profile for profiling release builds without losing symbols
[profile.profiling]
inherits = "release"
debug = true
```

## Algorithms And Data Structures

- Pick the right collection: `Vec` by default; `HashMap`/`HashSet` for lookup; `BTreeMap` for ordered keys and range queries; `VecDeque` for queues; `BinaryHeap` for priority queues.
- Replace repeated `Vec::contains` or linear searches with a set or map.
- Sort once, then use `binary_search` or dedup, not repeated scans.
- Batch I/O and database calls. An N+1 query pattern dwarfs any micro-optimisation.

## Allocation

```rust
use std::fmt::Write;

// ✅ Reserve when the size is known or estimable
fn squares(n: usize) -> Vec<u64> {
    let mut out = Vec::with_capacity(n);
    for i in 0..n as u64 {
        out.push(i * i);
    }
    out
}

// ✅ Build strings with write! into one buffer instead of many format! temporaries
fn csv_line(values: &[u32]) -> String {
    let mut line = String::with_capacity(values.len() * 4);
    for (i, value) in values.iter().enumerate() {
        if i > 0 {
            line.push(',');
        }
        write!(line, "{value}").unwrap(); // writing to a String cannot fail
    }
    line
}

// ✅ Reuse a buffer across iterations
fn longest_line(lines: &[&str]) -> usize {
    let mut buffer = String::new();
    let mut longest = 0;
    for line in lines {
        buffer.clear();
        buffer.push_str(line.trim());
        longest = longest.max(buffer.len());
    }
    longest
}

// ❌ Allocates per element just to look at it
// let upper: Vec<String> = names.iter().map(|n| n.to_uppercase()).collect();
// for n in &upper { ... }

fn main() {
    assert_eq!(squares(4), vec![0, 1, 4, 9]);
    assert_eq!(csv_line(&[1, 22, 333]), "1,22,333");
    assert_eq!(longest_line(&[" a ", "abc"]), 3);
}
```

- Borrow instead of cloning: pass `&str`/`&[T]`, return iterators, store `Arc<str>` or `Box<str>` for immutable shared text.
- Use `Cow<'_, str>` for "usually unchanged" transformations.
- Small-size optimisation crates (`smallvec`, `arrayvec`, `compact_str`) help only where profiling shows many tiny allocations.
- Avoid `Box<dyn Trait>`/`Vec<Box<T>>` in hot loops when an enum or `Vec<T>` works. Contiguous data is cache-friendly.
- For short-lived, many-object workloads (parsers, compilers) consider an arena (`bumpalo`).

## Iterators And Bounds Checks

Iterator chains compile to tight loops with no per-element bounds checks. Index loops may keep them.

```rust
fn dot(a: &[f64], b: &[f64]) -> f64 {
    a.iter().zip(b).map(|(x, y)| x * y).sum()
}

// Process fixed-size chunks to help vectorisation
fn sum_chunks(values: &[u32]) -> u64 {
    let mut chunks = values.chunks_exact(4);
    let mut total: u64 = chunks.by_ref().map(|c| u64::from(c.iter().sum::<u32>())).sum();
    total += chunks.remainder().iter().map(|&v| u64::from(v)).sum::<u64>();
    total
}

fn main() {
    assert_eq!(dot(&[1.0, 2.0], &[3.0, 4.0]), 11.0);
    assert_eq!(sum_chunks(&[1, 2, 3, 4, 5, 6]), 21);
}
```

- Prefer `iter()` over `for i in 0..v.len() { v[i] }`.
- Use `Iterator::sum`, `fold`, `chunks_exact`, `windows` rather than manual indexing.
- Do not `collect()` to a `Vec` mid-pipeline unless you need random access or multiple passes.

## Hashing

- The default `HashMap` uses SipHash, which resists hash-flooding from untrusted keys. Keep it for anything keyed by user input.
- For internal, trusted keys (ids, small ints) a faster hasher (`ahash`, `rustc-hash`/`FxHashMap`) can help. Measure first.

## I/O

```rust
use std::io::{BufRead, BufReader, BufWriter, Cursor, Write};

// ✅ Buffered reads and writes: unbuffered File/TcpStream issues one syscall per call
fn uppercase_lines<R: BufRead, W: Write>(input: R, output: W) -> std::io::Result<usize> {
    let mut out = BufWriter::new(output);
    let mut count = 0;
    for line in input.lines() {
        writeln!(out, "{}", line?.to_uppercase())?;
        count += 1;
    }
    out.flush()?;
    Ok(count)
}

fn main() -> std::io::Result<()> {
    let mut sink = Vec::new();
    let n = uppercase_lines(BufReader::new(Cursor::new("a\nb\n")), &mut sink)?;
    assert_eq!(n, 2);
    assert_eq!(sink, b"A\nB\n");
    Ok(())
}
```

- Lock `stdout` once for heavy printing (`let mut out = std::io::stdout().lock();`).
- Stream large files; do not `read_to_string` multi-gigabyte inputs.
- In async code, buffer with `tokio::io::BufReader` / `BufWriter`.

## Concurrency And Parallelism

- Data-parallel CPU work: `rayon` (`par_iter()`); keep tasks coarse enough to amortise scheduling.
- Reduce lock contention: shorter critical sections, sharding (`dashmap`), message passing, or `RwLock` for read-heavy data. Atomics for counters.
- Do not share a `Mutex<HashMap>` across all request handlers if a per-shard or actor design removes the lock.

## Build Profile Settings

```toml
[profile.release]
opt-level = 3          # default; use "s"/"z" for size-sensitive targets
lto = "thin"           # "fat" for the last few percent at higher build cost
codegen-units = 1      # better optimisation, slower builds
panic = "abort"        # smaller binary, no unwinding
strip = "symbols"
```

- Use `target-cpu=native` only for binaries you build and run on the same machine; never for distributed artifacts.
- Consider PGO (`cargo-pgo`) and allocators (`mimalloc`, `jemalloc`) for long-running servers, after benchmarking.
- Watch compile time too: heavy generics and large proc-macro use slow every build. `cargo build --timings` shows where.

## Quick Checklist

- [ ] A benchmark or profile justifies each optimisation
- [ ] Release-mode numbers only; benchmarks live in `benches/`
- [ ] No per-item allocation or clone in hot paths without reason
- [ ] I/O is buffered; database access is batched
- [ ] Async code never blocks the runtime
