# Rust Async And Concurrency

Examples use [tokio](https://tokio.rs), the de facto runtime for network services. Async Rust is lazy: a future does nothing until it is awaited or spawned.

## Choosing Concurrency

| Workload | Use |
|----------|-----|
| Many concurrent I/O waits (sockets, HTTP, DB) | `async` + tokio |
| CPU-bound work across cores | threads, `rayon`, or `spawn_blocking` from async code |
| Blocking library call inside async code | `tokio::task::spawn_blocking` |
| Simple script or CLI doing one thing at a time | plain synchronous code |

Do not make code `async` just because a library offers it. Async adds complexity (`Send` bounds, cancellation, pinning).

## Never Block The Runtime

An `async` task that blocks a worker thread stalls every other task on it.

```rust,ignore
// ❌ Blocks a runtime thread
async fn bad(path: &str) -> std::io::Result<String> {
    std::thread::sleep(std::time::Duration::from_secs(1)); // use tokio::time::sleep
    std::fs::read_to_string(path)                           // use tokio::fs or spawn_blocking
}

// ✅ Async APIs for I/O and timers
async fn good(path: &str) -> std::io::Result<String> {
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;
    tokio::fs::read_to_string(path).await
}

// ✅ CPU-heavy or blocking-only work on the blocking pool
async fn hash_password(password: String) -> Result<String, tokio::task::JoinError> {
    tokio::task::spawn_blocking(move || expensive_hash(&password)).await
}
```

- Rule of thumb: anything that can take more than ~100 µs without hitting an `.await` belongs on `spawn_blocking` or a dedicated thread pool.
- Never call `Runtime::block_on` or `futures::executor::block_on` from inside async code.

## Spawning And Structured Concurrency

```rust,ignore
use tokio::task::JoinSet;

// ✅ JoinSet owns its tasks: dropping it aborts them, results are collected
async fn fetch_all(urls: Vec<String>) -> Vec<Result<String, FetchError>> {
    let mut set = JoinSet::new();
    for url in urls {
        set.spawn(async move { fetch(&url).await });
    }

    let mut results = Vec::new();
    while let Some(joined) = set.join_next().await {
        match joined {
            Ok(result) => results.push(result),
            Err(join_err) => tracing::error!(%join_err, "task panicked or was cancelled"),
        }
    }
    results
}

// ✅ Run a fixed set concurrently on the current task; no spawn needed
async fn dashboard() -> Result<(Profile, Orders), ApiError> {
    tokio::try_join!(load_profile(), load_orders())
}
```

- `tokio::spawn` requires `'static + Send` futures: move owned data (or `Arc`) in; do not borrow locals.
- A dropped `JoinHandle` detaches the task; it keeps running. Keep handles or use a `JoinSet` so failures are observed.
- Bound concurrency with a `Semaphore` or `buffer_unordered(n)`. Never spawn one task per item of an unbounded input.

```rust,ignore
use std::sync::Arc;
use tokio::sync::Semaphore;

let permits = Arc::new(Semaphore::new(16));
for item in items {
    let permit = permits.clone().acquire_owned().await?;
    tokio::spawn(async move {
        let _permit = permit; // released on drop
        process(item).await;
    });
}
```

## Timeouts And Cancellation

Dropping a future cancels it at its next `.await`. Everything between two awaits runs to completion; everything after the last completed await is skipped.

```rust,ignore
use std::time::Duration;
use tokio::time::timeout;

// ✅ Every network call has a deadline
let body = timeout(Duration::from_secs(5), client.get(url).send())
    .await
    .map_err(|_| Error::Timeout)??;
```

### `select!` And Cancellation Safety

`tokio::select!` drops the losing branches. Only use branches whose futures are **cancellation safe** (dropping them loses no data). Cancel-safe: `mpsc::Receiver::recv`, `Sender::send`, `Mutex::lock`, `sleep`. Not cancel-safe: `read_exact`, `write_all`, and your own multi-step `async fn`s that mutate state between awaits.

```rust,ignore
loop {
    tokio::select! {
        // `recv` is cancel safe: no message is lost if another branch wins
        maybe_job = rx.recv() => {
            let Some(job) = maybe_job else { break };
            handle(job).await;
        }
        _ = shutdown.cancelled() => break,
        _ = tokio::time::sleep(Duration::from_secs(30)) => heartbeat().await,
    }
}
```

- Create a long-lived future once outside the loop and `&mut` it in `select!` if it must not restart each iteration (`tokio::pin!`).
- If an operation must finish once started, run it in its own spawned task and await the `JoinHandle`; the spawned task is not cancelled when the caller is.

## Locks Across `.await`

```rust,ignore
use std::sync::Mutex;

// ❌ std Mutex guard held across an await: future is !Send, can deadlock
async fn bad(state: &Mutex<Vec<u32>>) {
    let mut guard = state.lock().unwrap();
    guard.push(fetch_number().await);
}

// ✅ Do the await first, lock briefly afterwards
async fn good(state: &Mutex<Vec<u32>>) {
    let n = fetch_number().await;
    state.lock().unwrap().push(n);
}
```

- A `std::sync::Mutex` is fine in async code when the critical section is short and contains no `.await`. Use `tokio::sync::Mutex` only when you must hold the lock across an `.await`.
- Often better than any lock: give one task ownership of the state and talk to it through a channel (actor style).
- Clippy flags the bad pattern with `clippy::await_holding_lock` and `await_holding_refcell_ref`.

## Channels

| Channel | Use |
|---------|-----|
| `mpsc` (bounded) | Work queues, many producers to one consumer |
| `oneshot` | A single reply (request/response) |
| `broadcast` | Fan-out of events to many consumers |
| `watch` | Latest-value state (config reload, shutdown flag) |

```rust,ignore
use tokio::sync::{mpsc, oneshot};

enum Command {
    Get { key: String, reply: oneshot::Sender<Option<String>> },
    Set { key: String, value: String },
}

// Actor: one task owns the HashMap; no locks needed
async fn store(mut rx: mpsc::Receiver<Command>) {
    let mut data = std::collections::HashMap::new();
    while let Some(cmd) = rx.recv().await {
        match cmd {
            Command::Get { key, reply } => {
                let _ = reply.send(data.get(&key).cloned());
            }
            Command::Set { key, value } => {
                data.insert(key, value);
            }
        }
    }
}

let (tx, rx) = mpsc::channel(256); // always bounded: gives back-pressure
tokio::spawn(store(rx));
```

- Prefer **bounded** channels. An unbounded channel turns a slow consumer into unbounded memory growth.
- Decide what happens when the queue is full: await (back-pressure), `try_send` and drop or return an error.

## Graceful Shutdown

```rust,ignore
use tokio_util::sync::CancellationToken;
use tokio_util::task::TaskTracker;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let token = CancellationToken::new();
    let tracker = TaskTracker::new();

    for id in 0..4 {
        let token = token.child_token();
        tracker.spawn(async move { worker(id, token).await });
    }
    tracker.close();

    tokio::signal::ctrl_c().await?;
    token.cancel();                       // tell workers to stop
    tracker.wait().await;                 // let them finish in-flight work
    Ok(())
}
```

- Handle `SIGTERM` as well as Ctrl-C in containers (`tokio::signal::unix::signal(SignalKind::terminate())`).
- Give shutdown a deadline (`timeout(grace_period, tracker.wait())`) so a stuck task cannot block the deploy.

## Async Closures (Stable Since Rust 1.85)

```rust
// `async || ...` can borrow from its environment and works with AsyncFn bounds
async fn apply_twice<F>(f: F) -> u32
where
    F: AsyncFn(u32) -> u32,
{
    let once = f(1).await;
    f(once).await
}

async fn demo() -> u32 {
    let offset = 10;
    apply_twice(async |x| x + offset).await
}
```

## Observability

- Instrument with `tracing` (`#[tracing::instrument(skip(db))]` on async fns) instead of `println!`. Spans follow tasks across threads.
- Record task panics and cancellations; a silently dropped task is a silent failure.
- `tokio-console` helps find tasks that never wake or block the runtime.

## Quick Checklist

- [ ] No blocking calls (`std::fs`, `std::thread::sleep`, heavy CPU) on runtime threads
- [ ] Every network or external call has a timeout
- [ ] `select!` branches are cancellation safe or restructured
- [ ] No `std` lock guard is held across `.await`
- [ ] Channels are bounded; concurrency is limited by a semaphore or buffer size
- [ ] Spawned tasks are tracked (`JoinSet` / `TaskTracker`) and shut down gracefully
