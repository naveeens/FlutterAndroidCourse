# Intents

An `Intent` is Android's message object for "do this" or "here's data" — it can start an `Activity`, deliver a broadcast, or start a service, and carries a target (explicit: a specific class; implicit: an action the system resolves to whatever app handles it) plus a bundle of key/value extras.

## In the nested app

**Explicit intents**, targeting a specific class directly, are how this app's widget wires its own components together — `NestedWidgetProvider.kt` building the "open app" tap target:

```kotlin
val openAppIntent = Intent(context, MainActivity::class.java).apply {
    putExtra(EXTRA_PARENT_ID, state.currentFolderId)
    addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
}
```

`FLAG_ACTIVITY_NEW_TASK` is required here because the intent originates from outside any activity context (a `PendingIntent` fired by the launcher process, not from within a running `Activity`) — starting an activity that way requires explicitly creating a new task.

**Implicit intents** — where you declare what you *can* handle rather than what specifically launches you — are how this app receives Android's Share Sheet content. `AndroidManifest.xml` declares `ShareTargetActivity` as a handler for `ACTION_SEND` (plain text) and `ACTION_PROCESS_TEXT` (the text-selection "Process Text" menu):

```xml
<activity android:name=".share.ShareTargetActivity" android:exported="true" ...>
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.PROCESS_TEXT" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
</activity>
```

Any app on the device that shares plain text (a browser, another notes app) can now offer "Nested" as a share target, without either app knowing about the other in advance — that's the whole point of implicit intents. `ShareTargetActivity.kt` then reads whichever intent shape actually arrived:

```kotlin
private fun extractSharedText(intent: Intent?): String? {
    if (intent == null) return null
    return when (intent.action) {
        Intent.ACTION_SEND -> if (intent.type == "text/plain") intent.getStringExtra(Intent.EXTRA_TEXT) else null
        Intent.ACTION_PROCESS_TEXT -> intent.getCharSequenceExtra(Intent.EXTRA_PROCESS_TEXT)?.toString()
        else -> null
    }
}
```

**`onNewIntent`** handles the case where an already-running activity receives a *new* intent rather than being freshly launched — `MainActivity.kt`'s `launchMode="singleTop"` (set in the manifest) means a widget tap while the app is already open reuses the existing instance instead of creating a new one, routed through `onNewIntent`:

```kotlin
override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    setIntent(intent)
    extractFolderId(intent)?.let {
        pendingFolderId = it
        widgetNavChannel?.invokeMethod("onOpenFolder", it)
    }
}
```

This is what feeds `WidgetNavService`'s stream on the Dart side (see [[Streams]]) — a hot-start widget tap becomes a native `onNewIntent` call, forwarded across the platform channel into a Dart `Stream` event.

## Key takeaways

- Explicit intents (`Intent(context, TargetClass::class.java)`) for your own app's components; implicit intents (action + data type, resolved by the system) for interoperating with other apps — `ShareTargetActivity`'s manifest `intent-filter`s are what make this app a share target for any other app on the device.
- `onCreate` handles the intent an activity was *launched* with; `onNewIntent` handles one delivered to an *already-running* instance — required whenever `launchMode` isn't the default (`singleTop` here), as it is for this app's `MainActivity`.
- `addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)` is required when starting an activity from a context that isn't itself an activity (a broadcast receiver, a `PendingIntent` fired by another process) — as seen opening the app from the widget's header tap.
- See [[PendingIntent]] for how an `Intent` gets handed to another process to fire later, and [[Broadcast Receivers]] for the receiving side of this app's widget-tap intents.
