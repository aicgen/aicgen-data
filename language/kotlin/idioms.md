# Kotlin Idioms

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
