# Screenshots Tests

When working with UI screens, ALWAYS create Paparazzi Unit tests for them.

Paparazzi renders your screen into an image.

Here is an example of a Paparazzi test for a BookShelf screen:

```kotlin
class BookShelfPaparazziTest {

    @get:Rule
    val paparazzi = Paparazzi(
        deviceConfig = DeviceConfig.PIXEL_6,
        theme = "android:Theme.Material.Light.NoActionBar",
    )

    @Test
    fun bookShelfScreenContent() {
        paparazzi.snapshot {
            AppTheme {
                Box(
                    modifier = Modifier
                        .background(MaterialTheme.colorScheme.surface)
                        .padding(horizontal = 16.dp, vertical = 8.dp),
                ) {
                    BookShelfScreenContent(
                        modifier = Modifier.fillMaxSize(),
                        screenState = BookShelfScreenState(),
                        onAddFromFileClick = {},
                        onAddFromScanClick = {},
                        onSearchQueryChange = {},
                        onSearchMicClick = {},
                    )
                }
            }
        }
    }
}
```

This test should be located in `pro.labster.bookreader.ui.${ScreenPackage}.${ScreenName}PaparazziTest` file, e.g.
`pro.labster.bookreader.ui.book.shelf.BookShelfPaparazziTest`.

You should use `${ScreenName}ScreenContent` function there, e.g. `BookShelfScreenContent`.

*Don't forget to adjust the padding values if needed.*

Then, call `./gradlew recordPaparazziDebug` to record a snapshot of the screen.

After that, read the `${ProjectRoot}/app/src/test/snapshots/images/${TestPackageName}_${TestFileName}_${TestName}.png`, e.g.
`${ProjectRoot}/app/src/test/snapshots/images/pro.labster.bookreader.ui.book.shelf_BookShelfPaparazziTest_bookShelfScreenContent.png`.

If you were provided with a reference design image, compare them (always compare only the elements that are related to
the given task), and fix an issue if any, after that repeat testing.

If you don't have a reference design image, just check if the resulting image matches the description.

Please note that the resulting screenshot will only contain the rendered screen, e.g. it won't (most probably) contain
the navigation bar, status bar, etc.

## Miscellaneous

If a color doesn't exactly match (and no matching color exists in the theme), **DO NOT** specify it manually. Use the most
close color from the theme or explicitly ask the user. Always prefer the theme colors over manually specified colors.