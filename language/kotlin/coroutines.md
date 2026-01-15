# Kotlin Coroutines

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
