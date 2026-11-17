# Kotlin Idioms

## Rules

- Use scope functions for clarity, not to chain deeply: `let` for null checks and mapping, `apply` for configuring an object, `run` for computing a result, `also` for side effects.
- Prefer extension functions for small utilities, and expression bodies for simple functions.
- Use default and named arguments instead of overloads, and `sealed` types with exhaustive `when`.
- Avoid nesting scope functions more than one level; readability beats brevity.

## Scope Functions

Functions that execute a block of code within the context of an object.

### let

Use for null checks or mapping.

```kotlin
val str: String? = "hello"
str?.let {
    println(it.length)
}
```

### apply

Use for object configuration. Returns the object itself.

```kotlin
val dialog = Dialog(context).apply {
    setTitle("Hello")
    show()
}
```

### run

Use for object configuration and computing a result.

```kotlin
val result = service.run {
    configure()
    execute()
}
```

## Extension Functions

Add functionality to existing classes without inheriting from them.

```kotlin
fun String.removeSpaces(): String = this.replace(" ", "")

val clean = "hello world".removeSpaces()
```
