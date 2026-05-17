# Why is this `suspend` and that `launch`?

## The bug

A few months ago, while testing our messaging feature, I noticed that creating a new conversation did not always navigate into it. Sometimes it worked. Sometimes you tapped "Create" and ended up looking at an empty conversation list with no sign anything had happened. It was inconsistent and very hard to reproduce on a fast emulator.

The repository function looked like this:

```kotlin
fun createConversation(uuid: String, selectedIds: Set<String>, name: String?) = ioScope.launch {
    conversationDao.insert(ConversationEntity(uuid, name, ...))
}
```

And the ViewModel called it like this:

```kotlin
viewModelScope.launch {
    repository.createConversation(conversationId, selectedIds, name)
    openConversation(conversationId)
}
```

The fix was a one-line change to the repository function. Take a look at the diff and see if you can spot it:

```diff
-fun createConversation(uuid: String, selectedIds: Set<String>, name: String?) = ioScope.launch {
+suspend fun createConversation(uuid: String, selectedIds: Set<String>, name: String?) = withContext(Dispatchers.IO) {
```

## What was happening

`ioScope.launch { ... }` returns a `Job` immediately. The lambda runs concurrently on a background thread, and the caller has no built-in signal for when it finished. So `repository.createConversation(...)` came back instantly, and `openConversation(conversationId)` ran on the very next line, before the insert had finished. The conversation row did not exist yet. The navigation looked up a conversation that was not there.

On a slow device this happened often. On a fast emulator the insert occasionally won the race and everything looked fine, which is exactly the worst kind of bug.

`suspend fun + withContext(Dispatchers.IO)` is the opposite. You can only call it from inside a coroutine, and the calling coroutine is suspended until the work is done. By the time control returns to the next line, the work is guaranteed to have finished. Insert, then navigate. Sequential by default.

## Why some functions stay as `launch`

A few lines further down in the same repository there is `deleteMessage`, and it is still fire-and-forget:

```kotlin
fun deleteMessage(messageId: String) = ioScope.launch {
    messageDao.softDelete(messageId, Instant.now().toString())
}
```

Nothing the caller does next depends on the delete having finished. The UI will pick up the change the next time the Flow emits. Fire-and-forget is fine here, and making it `suspend` would force every caller to be in a coroutine for no real benefit.

## The question to ask

The decision is not arbitrary. Before reaching for one or the other, ask:

> Does the caller need to know this finished, or will something downstream break if it has not?

If yes, use `suspend`. If no, fire-and-forget is fine and saves the caller from extra ceremony.

## What I took from this

`launch` does not just control where the work runs, it changes the contract with the caller. A `suspend fun` is a promise that the work is done when control returns. A `fun = launch { ... }` is a promise that the work has started. Two very different promises. I had been treating them as interchangeable ways to "do this on a background thread", and that is what caused the bug.

The next post will probably be about `viewModelScope` itself, and why putting `launch` inside a ViewModel is safer than reaching for `GlobalScope`.
