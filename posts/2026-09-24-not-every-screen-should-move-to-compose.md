# Not every screen should move to Compose

## The gotcha

Most of the UI in the app I work on is moving to Jetpack Compose one piece at a time. It's easy to assume that ends with everything in Compose. I don't think it should, and the clearest case is a screen that's really a renderer, like a map or a game view.

Compose is built for UI that changes **when state changes**. A renderer changes **every frame**, whether anything happened or not.

## How Compose gets in

We don't convert whole screens. Compose comes in from the edges, as a View that the existing XML layout already knows how to place. The pattern we use most is a subclass of `AbstractComposeView`:

```kotlin
class ChatOverlay(
    context: Context,
    attrs: AttributeSet
) : AbstractComposeView(context, attrs) {
    private val visible = mutableStateOf(false)

    init {
        setViewCompositionStrategy(
            ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
        )
    }

    fun show() { visible.value = true }
    fun hide() { visible.value = false }

    @Composable
    override fun Content() {
        if (!visible.value) return
        AppTheme {
            ChatScreen()
        }
    }
}
```

It goes in the layout XML like any custom View. The legacy code around it doesn't need to know it's Compose: it calls `show()` and `hide()`, and those flip a piece of state that Compose is watching. The View finds the Activity's lifecycle through the view tree, so there's nothing else to wire up.

Dialogs are where that breaks. A dialog has its own window and its own view tree, so a `ComposeView` inside a plain `Dialog` can't find the Activity's lifecycle by walking up to it. Ours crashed with `ViewTreeLifecycleOwner not found` until we set all three owners on the dialog's decor view by hand:

```kotlin
dialog.window?.decorView?.let { decor ->
    decor.setViewTreeLifecycleOwner(activity)
    decor.setViewTreeViewModelStoreOwner(activity)
    decor.setViewTreeSavedStateRegistryOwner(activity)
}
```

If you can use `ComponentDialog` from androidx.activity instead, it sets the lifecycle and saved-state owners itself, but not the ViewModel store, so a `viewModel()` call inside it still needs the middle line.

## What suits Compose

The pieces that have moved are the ones that are a function of some state, like a list or a form. Nothing needs redrawing until something changes, and when something does, Compose only recomposes the parts that read it. Compared with inflating XML and updating Views by hand, that's a clear win.

## Why a renderer should stay a View

Say a screen is a map or a game view drawn with OpenGL. On Android that often means a `GLSurfaceView`, which renders on its own GL thread into its own surface, as often as every frame.

Moving that screen to Compose doesn't change any of it. Compose's drawing API has no way to make your own OpenGL calls, so GL in a Compose UI still renders into a separate surface. You'd host the same `GLSurfaceView` inside a composable (through `AndroidView`, for example), its GL thread would produce frames exactly as before, and it wouldn't draw any faster.

Compose's big advantage is that it only redoes the work that depends on what changed. On a screen where everything changes every frame, there's nothing for it to skip. For the effort you'd get the same renderer one layer deeper, and the risk of breaking a screen that already works.

## What migrating did change

The biggest surprise in the migration was about **when** things get created.

One of our settings pages used to be XML, inflated when the app started. Its repository registered a preferences listener, and because the page was always inflated, the listener was always there. Other code came to rely on that without anyone deciding it should: it wrote preferences at startup and counted on the listener to update the page's state.

In Compose, that page is only composed while its tab is open, and the listener's registration went with it. Until someone opened the tab, nothing was listening. The page's state went stale, and a guard that compared against the stale value made one of the dropdowns impossible to change. The fix was to stop depending on the timing: the setters now update the page's state themselves, and the listener is registered when the repository is created rather than when the page is shown.

That's the same shape as the [SharedPreferences post](2026-05-15-shared-prefs-listener.md): the listener works, and the assumption about **when** it's registered is wrong. XML inflated the page up front and hid that. In Compose, a tab's content only exists while the tab is showing. That's usually what you want, and it will find every place that depended on the old timing.

## Which screens to move

Almost any screen could be written in Compose. The better question for each one is whether Compose makes it easier to get right. For a settings form, it does. For a renderer that draws every frame on its own thread, it adds a layer and changes nothing that matters.
