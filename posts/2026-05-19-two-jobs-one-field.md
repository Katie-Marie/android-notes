# When a property does two unrelated jobs

## The gotcha

I was debugging a data corruption bug in an Android app. The fix turned out to be a structural refactor: one property in a data class had quietly been doing two unrelated jobs for years.

Here's the setup. A `Field` class:

```kotlin
class Field(
    val id: Int,
    val label: String,
    val data: Any?,
    val timestamp: Long = -1,
)
```

`timestamp` was used in two places:

**Job 1 (persistence):** the app uploads a CSV to a server. Each field gets a column for "when was this field last saved." That value comes from `field.timestamp`.

**Job 2 (UI lock):** for single-fill forms, a field becomes uneditable once it has been saved. The lock check:

```kotlin
val isFinal = mode == Single && field.timestamp >= 0
```

`timestamp >= 0` meant both "has been saved" AND "the UI must lock this." Same number, two meanings.

## The workaround that grew

These two purposes started fighting whenever the user was actively typing. We needed to update `timestamp` (so the new save time would persist) but we did NOT want to lock the field mid-edit. So a workaround appeared: a parallel list called `backingFields` that held the new timestamps separately from the displayed fields. The displayed field's `timestamp` stayed at the old value (UI did not lock); `backingFields[i].timestamp` got the new value (for the next save).

This dual-list design lived in the codebase for years. It was also the root cause of a race condition that produced cross-form data corruption: two mutable singleton properties that could get out of sync if the user navigated between forms at the wrong moment.

## The fix

Split the conflated responsibility:

```kotlin
class Field(
    val id: Int,
    val label: String,
    val data: Any?,
    val timestamp: Long = -1,         // persistence only
    val isEditable: Boolean = true,   // UI concern only
)
```

The UI lock check now reads `field.isEditable`. `timestamp` is free to update during typing without affecting the lock. The parallel list is gone. The race that depended on it is structurally impossible.

## Where this lives

The pattern to watch for: a workaround that feels disproportionate to its stated purpose. The dual-list approach was a lot of machinery (about 80 lines and a process-wide mutable singleton) to manage what felt like a small conflict. When you see this, the question to ask is: is one of these properties doing two unrelated jobs?

The diff to fix it is usually small. The downstream cleanup is large.
