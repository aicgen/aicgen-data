# Kotlin Coroutines

## Rules

- Use structured concurrency: launch coroutines in a lifecycle-bound scope (`viewModelScope`, `lifecycleScope`, `coroutineScope`), never `GlobalScope`.
- Make blocking work `suspend` and move it to the right dispatcher with `withContext`; inject dispatchers so tests can replace them.
- Never swallow `CancellationException`; rethrow it, and keep `try/catch` narrow around suspend calls.
- Use `async`/`await` for parallel results and `launch` for fire-and-forget work with error handling (`CoroutineExceptionHandler` or `supervisorScope`).
- Expose `Flow` for streams and collect it lifecycle-aware.

## Suspend Functions

Functions that can pause and resume execution without blocking the thread.

```kotlin
suspend fun fetchData(): String {
    delay(1000) // Non-blocking delay
    return "Data"
}
```

## Builders

- `launch`: Starts a new coroutine that does not return a result (fire and forget).
- `async`: Starts a new coroutine that returns a `Deferred` (like a Promise).

## Dispatchers

- `Dispatchers.Main`: UI thread.
- `Dispatchers.IO`: Network/Disk operations.
- `Dispatchers.Default`: CPU-intensive work.

```kotlin
viewModelScope.launch(Dispatchers.IO) {
    val data = fetchData()
    withContext(Dispatchers.Main) {
        updateUI(data)
    }
}
```
