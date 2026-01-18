# Architecture

This project is using the clean architecture approach.

```
// TODO:
// Add domain and UI models info
// Add mappers
```

## Interface/Implementation separation

Each class (except models = data classes) must be split into interface and implementation, e.g.:

```kotlin
interface BookRepository {
    // ...
}

internal class BookRepositoryImpl @Inject constructor(
    // ...
) : BookRepository {
    // ...
}
```

Both classes must be in the **same file**, e.g. `BookRepository.kt` in this case.

Implementation must ALWAYS be internal. Interface may be public or internal (all repositories and mappers must be
internal).

## Coroutines

We usually prefer functions that return `Flow` (even for a single item) over `suspend fun`.

## Layers

There are three layers:

- Data (`data` package)
- Domain (`domain` package)
- UI (Presentation, `ui` package)

### Data layer

Located in the `data` package.

Each feature has its own subpackage, e.g. `data.book` (singular, not plural).

Inside of it, we may have the following subpackages split by the data source (but not limited to):

- `db`
- `api`
- ...

For example: `data.book.db`

Each of them may have the following subpackages (but not limited to):

- `model`
- `mapper` (if we need to map data)
- `repository`
- ...

For example: `data.book.db.model`

For the database data source, we may have the following subpackages:

- `db` - Holds the Room database definition.
- `dao` - Holds the Room DAOs.

### Models

- Immutable data classes.
- May have computed properties (but don't forget to exclude them from serialization)
- For external APIs use KotlinX serialization models, and **always** specify `@SerialName` for each property.

For Room models:

- Always use `Long` for numeric PKs.
- Carefully design indexes where needed.
- Use type converters if you need to store complex data (e.g. KotlinX DateTime). Put converters in the `db.converter`
  package, one per file.

### Repositories

Stateless. May have `StateFlow`s in some cases (e.g. `SettingsRepository` should emit current setting value on change).

For Room database repositories:

- Prefer methods that return `Flow` over suspend functions.
- If we need to get value only once - use `.take(1)` on the DAO instead of observing.
- Always use `ioDispatcher` for Room database operations.

Naming convention: `${EntityName}${SourceName}Repository`, e.g. `BooksDbRepository` or `FavoritesApiRepository`.
Entity name is plural.

Example:

```kotlin
internal interface WatchfacesApiRepository {
    fun getWatchfaces(): Flow<List<WatchfaceApiModel>>
    fun getWatchface(slug: String): Flow<WatchfaceApiModel>

    class WatchfaceNotFoundException(
        val slug: String,
    ) : Exception("Watchface with slug $slug not found")
}

internal class WatchfacesApiRepositoryImpl @Inject constructor(
    private val httpClient: HttpClient,
    @DispatcherDefault
    private val defaultDispatcher: CoroutineDispatcher,
    @DispatcherIO
    private val ioDispatcher: CoroutineDispatcher,
) : WatchfacesApiRepository {

    override fun getWatchfaces(): Flow<List<WatchfaceApiModel>> {
        return typedFlow {
            // Call the external API
        }.flowOn(ioDispatcher) // Always set the dispatcher
    }

    override fun getWatchface(slug: String): Flow<WatchfaceApiModel> {
        return someApi
            .apiMethod()
            .flowOn(ioDispatcher) // Always set the corresponding dispatcher
            .map { watchfaces ->
                watchfaces.firstOrNull { it.slug == slug }
                    ?: throw WatchfacesApiRepository.WatchfaceNotFoundException(slug = slug) // Raise an exception
            }.flowOn(defaultDispatcher) // Always set the corresponding dispatcher
    }
}
```

Data layer must never be used directly from the UI layer.

### Domain layer

Located in the `domain` package.

Each feature has its own subpackage, e.g. `domain.book` (singular, not plural).

Generally, there are two types of entities on the domain layer:

- **Use case** – a class that performs a single action, stateless (except caching when needed)
- **Interactor** - a class that performs multiple actions, may be stateful

Each of them should be split into interface and implementation (in the same file).

Both of them are used in the `ui` layer.

### Use case

Use case performs a single action. It must be stateless (except caching when needed). It must implement a single
function - `oeperator fun invoke()`.

The file must be named `${ActionName}.kt`.

Example:

```kotlin
interface IsDebug {
    operator fun invoke(): Boolean
}

internal class IsDebugImpl @Inject constructor() : IsDebug {

    override fun invoke(): Boolean {
        return BuildConfig.DEBUG
    }
}
```

### Interactor

Interactors are more complex classes that perform multiple actions.

They implement business logic between `data` and `ui` layers.

In most cases, they are preferred over UseCases.

The file must be named `${Description}Interactor.kt`.

Example:

```kotlin
interface WearDevicesInteractor {
    val wearDevices: StateFlow<List<WearDeviceDomainModel>>
    val connectedNodes: StateFlow<List<Node>>

    fun setSelectedDeviceId(deviceId: String?): Flow<Unit>
}


internal class WearDevicesInteractorImpl @Inject constructor(
    private val wearDevicesRepository: WearDevicesRepository,
    @DispatcherDefault
    private val defaultDispatcher: CoroutineDispatcher,
) : WearDevicesInteractor {

    private val logger = getLogger(LOG_TAG)

    private val _wearDevices =
        MutableStateFlow<List<WearDeviceDomainModel>>(emptyList()) // Always create a mutable flow separately
    override val wearDevices = _wearDevices.asStateFlow()

    // Implementation

    private companion object {
        private const val LOG_TAG = "WearDevicesInteractor"
    }
}
```

## UI layer

Holds Compose screens, ViewModels and other UI-related classes.

ViewModels should use Interactors and UseCases from the Domain layer to perform actions.

Business logic should be kept in the Domain layer.

UI layer must never directly use the data layer entities (except for the models). Even if we need just to get some data
from repository without any changes, create a "proxy" function in the interactor.

See the [Screens](SCREENS.md) doc for more information about the UI layer.

## Layers dependencies

- UI layer depends on the Domain layer
- Domain layer depends on the Data layer
- Data layer depends on the external APIs, Room databases, Preferences and so on.

They **must not** depend on each other in any other direction, e.g. the Data layer should not depend on the domain
layer.