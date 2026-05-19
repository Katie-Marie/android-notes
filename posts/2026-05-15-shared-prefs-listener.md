# SharedPreferences listeners don't fire when you register them

## The gotcha

`SharedPreferences.OnSharedPreferenceChangeListener` fires on **changes**, not on registration. If you need the current value before any change happens, you have to read it explicitly. Easy to miss when you only test the "after a change" flow.

I missed this myself while wiring some UI state to a user setting. It looked fine until a clean install, when the UI sat on the default value because the listener never had anything to fire on. Fix in the immediate code: read the initial value explicitly in the init block before registering the listener.

## A nicer pattern

You can wrap both steps in a `Flow` that emits the initial value, then emits on changes. The gotcha gets handled inside the helper; callers just subscribe.

```kotlin
fun SharedPreferences.booleanFlow(key: String, default: Boolean): Flow<Boolean> = callbackFlow {
    // Emit current value first
    trySend(getBoolean(key, default))

    // Then emit on changes
    val listener = SharedPreferences.OnSharedPreferenceChangeListener { _, changedKey ->
        if (changedKey == key) {
            trySend(getBoolean(key, default))
        }
    }
    registerOnSharedPreferenceChangeListener(listener)

    awaitClose { unregisterOnSharedPreferenceChangeListener(listener) }
}
```

Caller code becomes one expression:

```kotlin
prefs.booleanFlow(KEY, false)
    .onEach { updateUI(it) }
    .launchIn(viewModelScope)
```

Multiple views can share the same Flow, so the gotcha is handled once at the source.

## Where this lives

The old way populated the state from the `ViewState` exposed by the `ViewModel`. This pattern centralises the SharedPreferences read in the data layer where it belongs. That data-layer placement is its own post (coming).
