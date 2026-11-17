# Kotlin Tooling And Project Setup

## Gradle Setup

- Use the Gradle wrapper, the **Kotlin DSL** and a **version catalog** (`gradle/libs.versions.toml`) so versions live in one place; pin the JDK with a toolchain.
- Fail the build on compiler warnings for new code and use `explicitApi()` in libraries.

```kotlin
// build.gradle.kts
plugins {
    alias(libs.plugins.kotlin.jvm)
    alias(libs.plugins.detekt)
    alias(libs.plugins.spotless)
}

kotlin {
    jvmToolchain(21)
    explicitApi()                      // libraries only
    compilerOptions { allWarningsAsErrors.set(true) }
}

tasks.test { useJUnitPlatform() }
```

## The Required Gate

```bash
./gradlew spotlessCheck detekt test
./gradlew dependencyUpdates          # or Renovate/Dependabot
```

| Concern | Tool |
|---------|------|
| Formatting | ktlint via Spotless, or ktfmt |
| Static analysis | detekt (with a committed baseline for legacy code only) |
| Tests | JUnit 5 + Kotest assertions; MockK; Turbine for Flow; `kotlinx-coroutines-test` |
| Serialization | kotlinx.serialization (compile-time, no reflection) |
| Dependencies | version catalog, dependency locking, OWASP Dependency-Check |

## Idioms

```kotlin
sealed interface Result<out T> {
    data class Ok<T>(val value: T) : Result<T>
    data class Err(val error: DomainError) : Result<Nothing>
}

enum class DomainError { NotFound, Invalid }

@JvmInline
value class UserId(val value: Long)

fun describe(result: Result<String>): String = when (result) {   // exhaustive: no else needed
    is Result.Ok -> "ok: ${result.value}"
    is Result.Err -> "error: ${result.error}"
}

data class Settings(val retries: Int = 3, val timeoutMs: Long = 30_000)

fun main() {
    println(describe(Result.Ok("done")))
    println(Settings().copy(retries = 5))
    println(UserId(7))
}
```

- Prefer `val`, immutable collections, `data class` for values, `sealed` types for state and results, and `value class` for typed ids.
- Model nullability in types; avoid `!!`. Use `requireNotNull`/`checkNotNull` with a message for invariants and `?:` for defaults.
- Use scope functions (`let`, `apply`, `also`) for clarity, not nesting. Prefer extension functions over utility classes.
- Coroutines: structured concurrency (`coroutineScope`, `supervisorScope`), never `GlobalScope`; inject dispatchers; make blocking calls inside `withContext(Dispatchers.IO)`; rethrow `CancellationException`; use `Flow` for streams with explicit `catch`/`flowOn`.
- Throw exceptions for programmer errors and use sealed results (or `Result`) for expected failures at boundaries.

## Testing

- Keep tests fast and isolated; test suspending code with `runTest`; control time with `TestDispatcher` rather than `delay`.
- Use real implementations or fakes for owned code; MockK for external boundaries.

## Quick Checklist

- [ ] Gradle wrapper, version catalog and JDK toolchain pinned
- [ ] Warnings are errors; format, detekt and tests run in CI
- [ ] No `!!`, no `GlobalScope`; cancellation is respected
