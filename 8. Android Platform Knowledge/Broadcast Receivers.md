# Broadcast Receivers

A `BroadcastReceiver` is a short-lived component that handles one `Intent` and then is gone — no persistent UI, no long-running process, just `onReceive(context, intent)` running briefly (a few seconds at most before the system may kill it) in response to a system or app-sent broadcast. They're the standard way something *outside* your running app (the launcher, in this app's case) triggers a small, self-contained piece of your code.

## In the nested app

Both of this app's custom receivers exist specifically to handle taps on the home-screen widget — see [[Widgets (App Widgets)]] for the full picture; this page is about the receiver mechanics themselves.

`WidgetRowActionReceiver.kt` handles taps inside the task list (checkbox toggle or row edit), dispatched via a merged [[PendingIntent]] + fill-in intent:

```kotlin
class WidgetRowActionReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val appWidgetId = intent.getIntExtra(NestedWidgetProvider.EXTRA_APPWIDGET_ID, -1)
        val taskId = intent.getLongExtra(NestedWidgetProvider.EXTRA_TASK_ID, -1L)
        if (taskId == -1L) return

        when (intent.getStringExtra(NestedWidgetProvider.EXTRA_ROW_ACTION)) {
            "toggle" -> {
                NodeDbHelper(context).toggleTaskCompletion(taskId)
                if (appWidgetId != -1) NestedWidgetProvider.refreshWidget(context, appWidgetId)
            }
            "edit" -> {
                context.startActivity(Intent(context, EditTaskActivity::class.java).apply { ... })
            }
        }
    }
}
```

This is a compact, complete example of what a receiver should do: read what it needs out of the `Intent`'s extras, do a small, fast piece of work synchronously (`NodeDbHelper.toggleTaskCompletion` is a direct SQLite `UPDATE`, not an async operation — see [[SQLite]]), and finish. There's no `Thread`, no coroutine, no waiting — exactly what `onReceive`'s tight execution window expects.

`WidgetNavReceiver.kt` is even simpler — it just advances which pinned folder the widget is currently showing, then triggers a refresh:

```kotlin
class WidgetNavReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val appWidgetId = intent.getIntExtra(NestedWidgetProvider.EXTRA_APPWIDGET_ID, -1)
        val direction = intent.getIntExtra(NestedWidgetProvider.EXTRA_DIRECTION, 0)
        if (appWidgetId == -1 || direction == 0) return
        WidgetPageResolver.advance(context, appWidgetId, direction)
        NestedWidgetProvider.refreshWidget(context, appWidgetId)
    }
}
```

Both are registered in `AndroidManifest.xml` as `exported="false"` — they're only ever triggered by `PendingIntent`s this app itself creates, never meant to be invokable by another app on the device, which `exported="false"` enforces at the system level.

`NestedWidgetProvider` itself (see [[Widgets (App Widgets)]]) is also, structurally, a `BroadcastReceiver` — `AppWidgetProvider` is a subclass that dispatches system widget-lifecycle broadcasts (`APPWIDGET_UPDATE`, etc.) into convenience methods like `onUpdate`.

## Key takeaways

- Keep `onReceive` fast and synchronous — a receiver has a strict, short execution budget before the system can consider it unresponsive; direct SQLite writes (as this app does) are fine, but anything genuinely slow belongs in [[WorkManager]] or a service instead.
- `exported="false"` on a receiver (both of this app's are) restricts it to intents from within the app itself — set `exported="true"` only when another app or the system genuinely needs to trigger it (as `NestedWidgetProvider` itself must be, to receive system widget broadcasts).
- Receivers are stateless between invocations — any state that needs to persist (like "which folder page is the widget on") has to live somewhere durable (`WidgetPrefs`'s `SharedPreferences`), not in receiver fields.
