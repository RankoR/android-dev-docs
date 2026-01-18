# Dependency Injection

Dependency injection is handled by Dagger/Hilt.

Files are located in the `di` package.

They should be split by feature.

There are two types of modules:

- `Binds`
- `Provides`

## Binds modules

Used when we want to provide a dependency using `@Binds` annotation. Preferred over `@Provides`.

File name: `${Feature}BindsModule.kt`.

Example:

```kotlin
@Module
@InstallIn(SingletonComponent::class)
internal abstract class WearBindsModule {

    @Binds
    @Reusable // Choose carefully
    abstract fun provideWearMessagingRepository(impl: WearMessagingRepositoryImpl): WearMessagingRepository
}
```

General function signature: `provide${DependencyName}(impl: ${DependencyName}Impl): ${DependencyName}`.

## Provides modules

Used when we need to execute some logic to provide a dependency.

File name: `${Feature}ProvidesModule.kt`.

Example:

```kotlin
@Module
@InstallIn(SingletonComponent::class)
internal class WearProvidesModule {

    @Provides
    @Reusable
    fun provideMessageClient(
        @ApplicationContext
        context: Context,
    ): MessageClient {
        return Wearable.getMessageClient(context)
    }
}
```

General function signature:

```
provide${DependencyName}(
    // Parameters, one per line
): ${DependencyName} {
    return // ...
}
```

## Scopes

Choose scopes carefully.

In general:

- `Interactor` should be `@Singleton`
- `UseCase` should be `@Singleton` if it has a state, and `@Reusable` otherwise
- `Repository` should be `@Reusable`