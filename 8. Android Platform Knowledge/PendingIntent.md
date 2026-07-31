# PendingIntent

A `PendingIntent` is a token that lets *another* process (the launcher, the notification system, `AlarmManager`) fire an `Intent` on your app's behalf, with your app's identity and permissions, at a later time — without that other process needing to actually construct or know the details of the `Intent` itself. It's the mechanism that makes [[Widgets (App Widgets)]] interaction possible at all, since a home-screen widget's taps are handled by the launcher process, which has no direct way to call into your code.

Two `PendingIntent` flags matter enough to always think about explicitly: **mutability** (`FLAG_MUTABLE` vs `FLAG_IMMUTABLE` — whether the intent's extras can be filled in later by the caller) and **request codes** (the integer that, combined with the intent's action/component/data, determines whether two `PendingIntent`s are considered "the same" and get merged, or distinct).

## In the nested app

`NestedWidgetProvider.kt` creates several `PendingIntent`s, and the differences between them are instructive. The prev/next navigation buttons each get their own broadcast `PendingIntent`, immediately actionable with no further data needed:

```kotlin
fun navPendingIntent(direction: Int): PendingIntent {
    val requestCode = appWidgetId * 10 + direction + 5
    val intent = Intent(context, WidgetNavReceiver::class.java).apply {
        action = ACTION_NAV
        putExtra(EXTRA_APPWIDGET_ID, appWidgetId)
        putExtra(EXTRA_DIRECTION, direction)
    }
    return PendingIntent.getBroadcast(context, requestCode, intent,
        PendingIntent.FLAG_MUTABLE or PendingIntent.FLAG_UPDATE_CURRENT)
}
```

`requestCode` is deliberately unique per widget instance *and* per direction (`appWidgetId * 10 + direction + 5`) — without that, Android would consider the prev and next buttons' `PendingIntent`s equivalent (same target, same action) and collapse them into one, breaking the ability to distinguish which arrow was tapped.

The task list's checkbox and title taps use a different mechanism — a *single* `PendingIntentTemplate` shared by every row, combined per-row with a `fillInIntent` (see `NestedRemoteViewsFactory.getViewAt` in `NestedRemoteViewsService.kt`):

```kotlin
val rowTemplate = PendingIntent.getBroadcast(context, appWidgetId,
    Intent(context, WidgetRowActionReceiver::class.java).setAction(ACTION_ROW),
    PendingIntent.FLAG_MUTABLE or PendingIntent.FLAG_UPDATE_CURRENT)
rv.setPendingIntentTemplate(R.id.widget_list, rowTemplate)
```

This is required by `ListView`/`RemoteViews`: creating a distinct real `PendingIntent` per row would be wasteful for a scrolling list, so instead one template is set on the list, and each row supplies only the *extras* that differ (`task_id`, which action) via `setOnClickFillInIntent` — the two get merged by the system at tap time. `FLAG_MUTABLE` is required here specifically because the template's extras genuinely need to be filled in later by each row.

Contrast that with the widget's title tap, which opens the app directly and needs no per-row merging, so it's `FLAG_IMMUTABLE`:

```kotlin
PendingIntent.getActivity(context, appWidgetId * 10 + 9, openAppIntent,
    PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT)
```

## Key takeaways

- `FLAG_MUTABLE` only when something else (a fill-in intent, per this app's row template) legitimately needs to modify the pending intent later — `FLAG_IMMUTABLE` otherwise, which is the safer default and required by newer Android versions for most cases that don't need mutability.
- Give each logically-distinct `PendingIntent` its own request code (as `navPendingIntent`'s `appWidgetId * 10 + direction + 5` does) — reusing one blindly across different intents is a common bug source where taps silently trigger the wrong action.
- `setPendingIntentTemplate` + `setOnClickFillInIntent` is the required pattern for per-row actions inside a widget's `ListView` — a single real `PendingIntent` can't scale to "one per row."
- `FLAG_UPDATE_CURRENT` ensures a re-created `PendingIntent` with the same request code refreshes its extras rather than silently keeping stale ones from a previous creation — used throughout this app's widget `PendingIntent`s.
