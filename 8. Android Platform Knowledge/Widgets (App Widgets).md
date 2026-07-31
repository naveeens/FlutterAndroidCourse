# Widgets (App Widgets)

An Android "app widget" (the home-screen kind, not a Flutter widget — confusing name collision worth flagging explicitly) is a piece of UI hosted by the *launcher* process, not your app. Your app never draws it directly; instead you describe it using `RemoteViews` (a serializable set of view-update instructions), and the launcher's process renders it. This is why app widgets can only use a constrained subset of Android views and why all interaction happens through `PendingIntent`s rather than normal click listeners — your code isn't running when the user taps the widget.

The nested app ships a real, fully-featured home-screen widget, making it one of the richest examples in this codebase of genuine native-Android (non-Flutter) engineering.

## The pieces, and how they fit together

**`AppWidgetProvider`** (`NestedWidgetProvider.kt`) is the entry point — a `BroadcastReceiver` subclass the system calls on widget lifecycle events (`onUpdate`, placed/removed, etc.). It builds the widget's *header* (title, prev/next arrows, add button) as `RemoteViews` and pushes it via `AppWidgetManager.updateAppWidget`:

```kotlin
private fun buildAndPushHeader(context: Context, mgr: AppWidgetManager, appWidgetId: Int) {
    val rv = RemoteViews(context.packageName, R.layout.widget_layout)
    val state = WidgetPageResolver.buildHeaderState(context, appWidgetId)
    ...
    rv.setTextViewText(R.id.widget_header_title, state.title)
    rv.setOnClickPendingIntent(R.id.widget_prev, navPendingIntent(-1))
    mgr.updateAppWidget(appWidgetId, rv)
}
```

**`RemoteViewsService` + `RemoteViewsFactory`** (`NestedRemoteViewsService.kt`) supplies the scrollable *list* of tasks inside the widget — `ListView`s in widgets can't just take a normal adapter, since (again) nothing runs in your process; the factory's `getViewAt(position)` builds one `RemoteViews` row at a time, on demand, when the launcher's remote adapter needs it:

```kotlin
override fun getViewAt(position: Int): RemoteViews {
    val task = tasks[position]
    val rv = RemoteViews(context.packageName, R.layout.widget_item)
    rv.setTextViewText(R.id.item_title, task.name)
    rv.setOnClickFillInIntent(R.id.item_check, Intent().apply {
        putExtra(NestedWidgetProvider.EXTRA_ROW_ACTION, "toggle")
        putExtra(NestedWidgetProvider.EXTRA_TASK_ID, task.id)
    })
    return rv
}
```

**Interaction** happens via [[Broadcast Receivers]] and [[PendingIntent]] — see those pages for the full mechanics. In short: tapping a checkbox or a title in the list fires a `fillInIntent` merged with the list's shared `PendingIntentTemplate`, landing in `WidgetRowActionReceiver`; tapping the prev/next arrows fires a separate `PendingIntent` straight to `WidgetNavReceiver`.

**Data** comes directly from the app's own SQLite file, read in-process by `NodeDbHelper.kt` — see [[Content Providers]] for why this app reads the `.db` file directly instead of exposing a `ContentProvider`.

**Refreshing** happens two ways: `notifyAppWidgetViewDataChanged(appWidgetId, R.id.widget_list)` tells the launcher to re-ask the `RemoteViewsFactory` for fresh rows (used after any data change — see `WidgetBridge.refreshWidget()` in `lib/services/widget_bridge.dart`, called from Dart via `MethodChannel` every time `NestedTreeNotifier` mutates something); `updateAppWidget(appWidgetId, rv)` (via `refreshWidget`) rebuilds the header itself (title, count, arrow enabled-state).

**Multi-page navigation** is state the widget itself has to track, since it's not backed by a live Flutter engine — `WidgetPageResolver.kt` is the "single source of truth for which page is currently showing," persisted per-widget-instance in `WidgetPrefs` (`SharedPreferences`, keyed by `appWidgetId` so multiple placed widgets can each show a different pinned folder independently).

**Configuration** lives in `res/xml/nested_widget_info.xml`:

```xml
<appwidget-provider
    android:minWidth="250dp" android:minHeight="180dp"
    android:targetCellWidth="4" android:targetCellHeight="3"
    android:updatePeriodMillis="0"
    android:initialLayout="@layout/widget_layout"
    android:resizeMode="horizontal|vertical"
    android:widgetCategory="home_screen" />
```

`updatePeriodMillis="0"` means the system's periodic auto-refresh is disabled entirely — this widget only refreshes when explicitly told to (via the Dart-to-native `refreshWidget()` bridge), not on a timer, since it always has fresh data available on demand rather than needing to poll.

## Key takeaways

- Widget UI is `RemoteViews`, not real views running in your process — every interaction must go through `PendingIntent`, and every visual update is pushed explicitly via `AppWidgetManager`.
- `AppWidgetProvider` (header/lifecycle) and `RemoteViewsFactory` (list rows) are separate pieces with separate refresh calls (`updateAppWidget` vs. `notifyAppWidgetViewDataChanged`) — this app's `refreshWidget()` calls both together since either header state or list content (or both) can change on any given data mutation.
- A widget instance persists state (like "which pinned folder page am I on") independently per `appWidgetId`, since there's no running app process to hold it in memory between interactions.
- See [[MethodChannel]] and [[Platform Channels]] for how `NestedTreeNotifier`'s writes on the Dart side trigger a widget refresh on the native side.
