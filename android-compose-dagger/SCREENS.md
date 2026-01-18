# This document describes the screen architecture

Each screen should be placed in a separate package inside the `ui` package.

Depending on the level, this package may be right in the `ui` package or in a parent feature subpackage.
For example:

- `ui.splash` - Splash screen feature holds only one screen, so the screen should be placed in the `ui` package.
- `ui.book.detail` - Book feature holds multiple screens, so they must be placed in the `ui.book` subpackage.

Each screen consists of these components:

- `ScreenState` - Screen's state.
- `ViewModel` - Screen's ViewModel.
- `Screen` - Compose implementation of the screen.

Each **feature** also has the code for navigation (see below).

## ScreenState

Should have name `${ScreenName}ScreenState`, e.g. `BookDetailScreenState`.
Located in the `model` subpackage, e.g. `ui.book.detail.model.BookDetailScreenState.kt`.

Consists of a single immutable data class (if needed - might have inner classes) holding all information needed to
render the screen. Emitted by the ViewModel.

ScreenState should have default values for all its properties.

Example:

```kotlin
@Immutable
internal data class BookDetailScreenState(
    val book: Book? = null,
    val bookState: BookState = BookState.Unknown,
    val bottomSheet: BottomSheet? = null,
) {

    @Immutable
    sealed interface BookState {
        object Unknown : BookState
        object NotDownloaded : BookState
        data class Downloading(
            val progress: Int,
        ) : BookState
        object Downloaded : BookState
    }

    // Use this if there are any bottom sheets on the screen
    @Immutable
    sealed interface BottomSheet {

        @Immutable
        data class DeleteBookBottomConfirmSheet(
            val book: Book,
        ) : BottomSheet
    }
}
```

## ViewModel

Should have name `${ScreenName}ViewModel`, e.g. `BookDetailViewModel`.
Located in the screen package, e.g. `ui.book.detail.BookDetailViewModel.kt`.

Consists of a single class holding all logic needed to control the screen. Emits `${ScreenName}ScreenState` and handles
user
interactions.

Template:

```kotlin
@HiltViewModel
internal class BookShelfViewModel @Inject constructor(
    resources: Resources,
    @DispatcherDefault
    defaultDispatcher: CoroutineDispatcher,
    @DispatcherIO
    ioDispatcher: CoroutineDispatcher,
    @DispatcherMain
    mainDispatcher: CoroutineDispatcher,
) : BaseViewModel<BookShelfScreenState>(
    resources = resources,
    defaultDispatcher = defaultDispatcher,
    ioDispatcher = ioDispatcher,
    mainDispatcher = mainDispatcher,
    defaultContentValue = BookShelfScreenState(),
) {

    override val logger = getLogger(LOG_TAG)

    private companion object {
        private const val LOG_TAG = "BookShelfViewModel"
    }
}
```

If screen doesn't have any state (e.g. the splash screen), don't create the ScreenState object and use
`BaseUnitViewModel` instead.

Template:

```kotlin
@HiltViewModel
internal class MainViewModel @Inject constructor(
    resources: Resources,
    @DispatcherDefault
    defaultDispatcher: CoroutineDispatcher,
    @DispatcherIO
    ioDispatcher: CoroutineDispatcher,
    @DispatcherMain
    mainDispatcher: CoroutineDispatcher,
) : BaseUnitViewModel(
    resources = resources,
    defaultDispatcher = defaultDispatcher,
    ioDispatcher = ioDispatcher,
    mainDispatcher = mainDispatcher,
) {
    override val logger = getLogger(LOG_TAG)


    private companion object {
        private const val LOG_TAG = "MainViewModel"
    }
}
```

### Modifying screen state in ViewModel

`BaseViewModel` and `BaseUnitViewModel` have useful methods for modifying screen state:

- `protected fun setScreenState(state: ScreenState)` - sets a new screen state directly.
- `protected fun setLoadingScreenState()` - sets the loading screen state.
- `protected fun setContentScreenState()` - sets the content screen state with the current `contentValue`.
- `protected fun updateAndShowContent(block: C.() -> C)` - updates the current content value (creating a copy
  internally), and changes screen state to content (if not yet).
- `protected suspend fun updateAndShowContentThreadSafe(block: C.() -> C)` - same as `updateAndShowContent`, but
  thread-safe (for handling possible race conditions while updating `contentValue`).
- `protected fun setErrorScreenState(message: String)` - sets the screen state to error with the given message. Should
  generally not be used (methods like `showErrorMessage` usually fit better).

In general, you should use `updateAndShowContent` method to modify screen state.

Example:

```kotlin
private fun loadWatchface(slug: String) {
    logger.d { "loadWatchface: slug=slug" }

    setLoadingScreenState()

    viewModelScope.launch(defaultDispatcher) {
        watchfacesInteractor
            .getWatchface(slug = slug)
            .onErrorShowMessage()
            .collect { watchface ->
                updateAndShowContent {
                    copy(
                        // <-- This is called on the current screen state object
                        watchface = watchface,
                    )
                }
            }
    }
}
```

### Showing messages from ViewModel

`BaseViewModel` and `BaseUnitViewModel` have useful methods for showing messages:

- `protected suspend fun showSnackBarMessage(message: String)` - shows a snack bar with the given message.
- `protected suspend fun showSnackBarMessage(@StringRes messageResourceId: Int)` - overload
- `protected suspend fun showMessageBottomSheet(message: String)` - shows a bottom sheet with the given message. Bottom
  sheet is handled automatically by the `BaseScreen`, should not be implemented in `ScreenState`.
- `protected suspend fun showMessageBottomSheet(@StringRes messageResourceId: Int)` - overload

### Handling and showing errors from ViewModel

`BaseViewModel` and `BaseUnitViewModel` have useful methods for showing error messages:

- `suspend fun showErrorMessage(message: String)` - shows an error message with the given message.
- `protected suspend fun showErrorBottomSheet(message: String)` - shows a bottom sheet with the given error message.
  Bottom sheet is handled automatically by the `BaseScreen`, should not be implemented in `ScreenState`.
- `protected suspend fun showErrorBottomSheet(@StringRes messageResourceId: Int)` - overload

They also have these useful extension functions for handling errors in `Flow`s:

```kotlin
protected fun <T> Flow<T>.onErrorShowMessage(
    getMessage: suspend (throwable: Throwable) -> String = { throwable ->
        throwable.message ?: throwable.toString()
    },
): Flow<T>
```

This method will catch any errors emitted by the flow and show an error message with the given message. Error will not
be propagated further.

```kotlin
protected fun <T> Flow<T>.onErrorShowMessageAnd(
    getMessage: (throwable: Throwable) -> String = { throwable ->
        throwable.message ?: throwable.toString()
    },
    block: () -> Unit,
): Flow<T>
```

Same as above, but also calls the given block when an error occurs.

So, in most cases when you're collecting a flow in a ViewModel, you should use these extension functions instead of
handling errors manually.

```kotlin
open fun onRetryClick()
```

This method will be called from the screen's error state. As the error state should not be generally used, this method
should not be overridden in the ViewModel.

### Navigation from the ViewModel

`BaseViewModel` and `BaseUnitViewModel` have these properties for navigation:

- `onNavigateTo: OnNavigateTo` - to navigate to another screen. Accepts nav destination and a block with actions to
  perform on navigation.
- `onPopBackStack: OnPopBackStack` - to pop the current screen from the back stack. No arguments.
- `onClearBackStack: OnClearBackStack` - to clear the back stack completely. No arguments. Should not be generally used,
  only in special cases.

Usage examples:

```kotlin
withContext(mainDispatcher) {
    onNavigateTo(MainScreenNavDestination.Store) {}
}
```

Or, with arguments:

```kotlin
withContext(mainDispatcher) {
    onNavigateTo(
        WatchfaceScreenNavDestination.Detail(
            slug = favoriteWatchface.watchface.slug
        ),
    ) {}
}
```

Or, to clear the back stack upon navigation:

```kotlin
withContext(mainDispatcher) {
    onNavigateTo(MainScreenNavDestination.Store) {
        onClearBackStack()
    }
}
```

## Screen

Should have name `${ScreenName}Screen`, e.g. `BookDetailScreen`.
Located in the screen package, e.g. `ui.book.detail.BookDetailScreen.kt`.

Consists of multiple Compose functions. Should expose only one Compose function named `${ScreenName}Screen`, e.g.
`BookDetailScreen`.

Template:

```kotlin
@Composable
internal fun WatchfaceDetailScreen(
    modifier: Modifier = Modifier,
    onNavigateTo: OnNavigateTo,
    onClearBackStack: OnClearBackStack,
    onPopBackStack: OnPopBackStack,
    slug: String,
    viewModel: WatchfaceDetailViewModel = hiltViewModel(),
) {
    // Setting nav functions to the ViewModel
    viewModel.onNavigateTo = onNavigateTo
    viewModel.onClearBackStack = onClearBackStack
    viewModel.onPopBackStack = onPopBackStack

    viewModel.setSlug(slug = slug) // If there is a navigation argument

    BaseScreen(
        viewModel = viewModel,
    ) { screenState -> // <-- This is called on every screen state change
        WatchfaceDetailScreenContent(
            modifier = modifier
                .fillMaxSize()
                .background(MaterialTheme.colorScheme.surface),
            screenState = screenState,
            onInstallClick = viewModel::onInstallClick,
            onPreviewClick = viewModel::onPreviewClick,
            onTagClick = viewModel::onTagClick,
        )

        ActiveBottomSheet(
            // If there are any bottom sheets on the screen (except the message/error bottom sheets that are handled automatically by the BaseScreen)
            data = screenState.bottomSheet,
            onDismissRequest = viewModel::onBottomSheetDismissRequest,
        ) { sheetData ->
            when (sheetData) {
                is WatchfaceDetailScreenState.BottomSheet.WatchfacesLimitDataSheet -> {
                    WatchfacesLimitSheetContent(
                        watchfacesToUninstall = sheetData.watchfacesToUninstall,
                        onWatchfaceUninstallClick = viewModel::onWatchfaceUninstallOnLimitClick,
                    )
                }
            }
        }
    }
}
```

So, the exposed function is just a "host" for the "content" function. Content function should be named
`${ScreenName}ScreenContent`.

Template:

```kotlin
@Composable
private fun WatchfaceDetailScreenContent(
    modifier: Modifier = Modifier,
    screenState: WatchfaceDetailScreenState,
    onInstallClick: () -> Unit,
    onPreviewClick: () -> Unit,
    onTagClick: (tag: String) -> Unit,
) {
    // Actual content here
}
```

Content function should be also split into multiple smaller functions for readability, if needed.

### Clean code

Generally, you should split the Content function into multiple smaller functions, e.g. by block.
Let's say we have:

- Search field
- Header
- List of items

In this case, we should have at least these separate compose functions for:

- Search field
- Header
- List of items
- Item of the list

Sometimes it's also nice to have an empty state function (e.g. if we have an empty list).

Each of them should have a preview (see below).

If there are many Compose functions in one file, consider moving them to separate files located in the `component`
subpackage (one component per file), e.g.:

- `ui.book.detail.component.SearchField.kt`
- `ui.book.detail.component.Header.kt`
- `ui.book.detail.component.ItemsList.kt` (should also contain the list item function and empty state function if
  applicable)

If putting components in separate files, make sure to add `@Preview` functions for each of them right in the same file.

### Screen preview

For each screen, there should be a preview generated in the same file (at the very bottom of the file).

Preview should use the **content** function, not the **screen** function.

Preview function should be named `${ScreenName}ScreenContentPreview`, e.g. `WatchfaceDetailScreenContentPreview`.
If we need multiple previews (e.g. for the different screen states), functions should be named
`${ScreenName}${ScreenStateDescription}ScreenContentPreview`, e.g. `WatchfaceDetailScreenContentNoItemsPreview`.

Template:

```kotlin
@Preview
@Composable
private fun WatchfaceDetailScreenContentPreview() {
    AppTheme {
        Box(
            modifier = Modifier
                .size(width = 400.dp, height = 800.dp)
                .background(MaterialTheme.colorScheme.surface),
        ) {
            WatchfaceDetailScreenContent(
                modifier = Modifier
                    .fillMaxSize(),
                screenState = WatchfaceDetailScreenState(
                    // Create the screen state with the data needed for the preview
                ),
                onInstallClick = {},
                onPreviewClick = {},
                onTagClick = {},
            )
        }
    }
}
```

For the individual parts of the screen, create the previews too (above the screen content preview).
Theys should be named `${FunctionName}Preview`, or `${FunctionName}${ComponentStateDescription}Preview` if needed, e.g.
`SearchFieldPreview` or `SearchFieldErrorPreview`.

## Bottom sheets

Besides the default bottom sheet handling provided by the `BaseScreen`, screen can also have its own custom bottom
sheets.

These bottom sheets are declared using `pro.labster.bookreader.ui.base.sheet.ActiveBottomSheetKt.ActiveBottomSheet`
composable function.

It has the following signature:

```kotlin
@Composable
fun <T : Any> ActiveBottomSheet(
    data: T? = null,
    shouldDismissOnBackPress: Boolean = true,
    shouldDismissOnClickOutside: Boolean = true,
    onDismissRequest: () -> Unit = {},
    content: @Composable (T) -> Unit,
)
```

See the usage example above in the screen example.
Bottom sheet state should be controlled by the ScreenState (see the example in the ScreenState example).

`onDismissRequest` should be handled by the ViewModel. If we want to dismiss the bottom sheet - set it to `null` in the
ScreenState.

## Navigation

Each feature declares its navigation in the `ui.feature.nav.NavDestination.kt` file, e.g.
`ui.book.nav.BookNavDestination.kt`.

Here is an example:

```kotlin
@Serializable
sealed interface WatchfaceScreenNavDestination : NavDestination {

    @Serializable
    data object Main : WatchfaceScreenNavDestination

    @Serializable
    data class Detail(
        @SerialName("slug")
        val slug: String,
    ) : WatchfaceScreenNavDestination
}

fun NavGraphBuilder.watchfaceScreen(
    onNavigateTo: OnNavigateTo,
    onClearBackStack: OnClearBackStack,
    onPopBackStack: OnPopBackStack,
) {

    composable<WatchfaceScreenNavDestination.Main> {
        WatchfaceMainScreen(
            // Composable function for the screen
            onNavigateTo = onNavigateTo,
            onClearBackStack = onClearBackStack,
            onPopBackStack = onPopBackStack,
        )
    }

    composable<WatchfaceScreenNavDestination.Detail> {
        WatchfaceDetailScreen(
            // Composable function for the screen
            onNavigateTo = onNavigateTo,
            onClearBackStack = onClearBackStack,
            onPopBackStack = onPopBackStack,
            slug = it.toRoute<WatchfaceScreenNavDestination.Detail>().slug,
        )
    }
}
```

If navigation destination has arguments, do NOT pass objects there, pass basic types instead - `Int`, `Long`, `String`
and so on.

Then they are included in the `pro.labster.bookreader.ui.nav.AppNavigationKt.AppNavigation` function:

```kotlin
@Composable
internal fun AppNavigation(
    // ...
) {
    // ...

    NavHost(
        // ...
    ) {
        // ...
        watchfaceScreen(
            onNavigateTo = defaultOnNavigateTo,
            onClearBackStack = defaultOnClearBackStack,
            onPopBackStack = onPopBackStack,
        )

        // ...
    }
}
```

## Icons

Prefer default Compose icons

## Haptic feedback

Many interactions (like clicking somewhere) should trigger haptic feedback. Use `LocalHapticFeedback` to get the
feedback instance.