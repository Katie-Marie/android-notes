# SharedPreferences listeners don't fire when you register them

## The gotcha

I added a `SharedPreferences.OnSharedPreferenceChangeListener` to keep some UI state in sync with a user setting. It seemed to work fine while testing it until I started with a clean install and realised it was not getting the initial value. That's because `OnSharedPreferenceChangeListener` is just that, for **changes**, and I needed to set the initial value in the init block.

## A nicer pattern

This all worked but there has to be a nicer way to initialise and subscribe to changes in the one place right? Well you can wrap the whole thing in a `Flow` that emits the initial value, then emits on changes. The gotcha gets handled inside the helper; callers just subscribe.

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

The beauty of this is that if you have multiple views using this same SharedPref you can reuse this.

## Where this lives

In the old way I was populating the state from the `ViewState` we expose via the `ViewModel`. This new way of doing things centralises the flow of `SharedPreferences` and puts them in the data layer. That's its own post though, probably one about how the View, ViewModel, and Data layers should connect.
