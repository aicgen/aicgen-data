# Kotlin Basics

## Rules

- Prefer `val` over `var` and immutable collections (`List`, `Map`) over mutable ones.
- Avoid `!!`; handle nullability with `?.`, `?:`, `let`, or an early return.
- Use data classes for value holders, sealed classes or interfaces for closed hierarchies, and `when` as an exhaustive expression.
- Follow the Kotlin coding conventions and run ktlint or detekt.

## Null Safety

Kotlin distinguishes between nullable and non-nullable types.

- `String` - Cannot hold null.
- `String?` - Can hold null.

```kotlin
var a: String = "abc"
// a = null // compilation error

var b: String? = "abc"
b = null // ok
```

Use safe calls (`?.`) or the Elvis operator (`?:`) when dealing with nullable types.

```kotlin
val l = b?.length ?: -1
```

## Val vs Var

- `val`: Read-only (immutable reference). Prefer this by default.
- `var`: Mutable.

## Data Classes

Use data classes to hold data. They automatically generate `equals()`, `hashCode()`, `toString()`, and `copy()`.

```kotlin
data class User(val name: String, val age: Int)
```
