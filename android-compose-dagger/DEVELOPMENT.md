# Development guidelines

## Tech stack

- **Kotlin** for the programming language.
- **Jetpack Compose** for the UI layer.
- **Hilt** for dependency injection.
- **Coroutines & Flow** for asynchronous programming.
- **Room** for local database.
- **DataStore** for preferences.
- **Coil** for image loading.
- **Kermit** for logging.
- **Kotlinx Serialization** for JSON parsing.

## Dependencies

Dependencies must be declared in the `gradle/libs.versions.toml` file.

## Architecture

Clean architecture.

See the [Architecture](ARCHITECTURE.md) page for the detailed information.

## DI

See the [Dependency Injection](DI.md) page for the detailed information.

Please note that the Coroutines dispatchers must always be injected using Dagger. We have the following qualifiers:

- `pro.labster.bookreader.di.qualifier.DispatcherIO`
- `pro.labster.bookreader.di.qualifier.DispatcherDefault`
- `pro.labster.bookreader.di.qualifier.DispatcherMain`

## UI layer

See the [Screens](SCREENS.md) page for the detailed information on how to implement screens.

## Logging

Use `pro.labster.bookreader.util.logging.LoggerUtilsKt.getLogger(tag: String)` function to get a logger for a class.

- `Kermit` is used under the hood.
- Tag must be specified as a private constant `LOG_TAG` in the companion object.
- Tag must reflect the class name, e.g. `FavoritesApiRepository` (but **NOT** `FavoritesApiRepositoryImpl`, even if it's declared inside the implementation class).
- Always prefer to log as much information as possible.
- When constructing a message, add the function name to it (and arguments if needed)

Example:

```kotlin
interface SomeRepository 

class SomeRepositoryImpl : SomeRepository {
    private val logger = getLogger(LOG_TAG)
    
    private fun doSomething(someArg: String) {
        logger.d("doSomething: someArg=$someArg")
        
        // ...
        
        logger.d("doSomething: done")
    }
    
    private companion object {
        private const val LOG_TAG = "SomeRepository"
    }
}
```

## Flow helpers

There are some helpers for working with Flows in the `pro.labster.bookreader.util.KotlinFlowExtensions.kt` file:

```kotlin
inline fun <T> Flow<T>.ignoreValue(): Flow<Unit> {
    return this.map { }
}

inline fun unitFlow(crossinline block: suspend FlowCollector<Unit>.() -> Unit): Flow<Unit> {
    return flow {
        block()
        emit(Unit)
    }
}

fun <T> errorFlow(throwable: Throwable): Flow<T> {
    return flow {
        throw throwable
    }
}

inline fun <T> typedFlow(crossinline block: suspend FlowCollector<T>.() -> T): Flow<T> {
    return flow {
        val value = block()

        emit(value)
    }
}

inline fun <T> optionalTypedFlow(crossinline block: suspend FlowCollector<T>.() -> T?): Flow<T> {
    return flow {
        val value = block()

        if (value != null) {
            emit(value)
        }
    }
}

// And others
```

## Testing

TODO

## Coding conventions

Generally, we're using the official Kotlin coding conventions.

### Functions

**NEVER** implement functions like this:

```kotlin
fun doSomething(): SomeReturnValue = someAction()
```

Instead, implement them like this:

```kotlin
fun doSomething(): SomeReturnValue {
    return someAction()
}
```

### Companion objects

Companion objects must be declared at the bottom of the class.

If a companion object doesn't have public methods, make it private explicitly.

### Misc

- If there is more than one argument, always place them on separate lines.
- Always use trailing commas.
- Always use named arguments.
- If a class isn't supposed to be used outside the module, make it internal.