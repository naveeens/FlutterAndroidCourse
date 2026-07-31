# Background execution

"Background execution" covers any work your app does when it's not the foreground, focused activity — and Android has progressively tightened what's allowed there (to save battery), which is why there's a whole family of purpose-built APIs ([[WorkManager]], [[Foreground services]], [[AlarmManager]]) rather than just "spawn a thread and let it run."

## What actually runs in the "background" in the nested app

This app has no long-running background *process* — no service that stays alive, no scheduled periodic work, no foreground notification. But it does have code that executes **without a running foreground `Activity`**, which is a related but distinct category worth separating out clearly: its `BroadcastReceiver`s (`WidgetRowActionReceiver`, `WidgetNavReceiver`) and `RemoteViewsService` (see [[Broadcast Receivers]], [[Widgets (App Widgets)]]) run whenever the user interacts with the home-screen widget, regardless of whether the app itself is open, backgrounded, or not running at all.

```kotlin
override fun onReceive(context: Context, intent: Intent) {
    val taskId = intent.getLongExtra(NestedWidgetProvider.EXTRA_TASK_ID, -1L)
    if (taskId == -1L) return
    when (intent.getStringExtra(NestedWidgetProvider.EXTRA_ROW_ACTION)) {
        "toggle" -> {
            NodeDbHelper(context).toggleTaskCompletion(taskId) // direct, synchronous SQLite write
            ...
        }
    }
}
```

This works without any of the heavier background-execution machinery specifically *because* it's small and synchronous — a receiver's `onReceive` runs on the main thread with a short execution budget (a few seconds), and a single `UPDATE` statement on a small local SQLite database comfortably fits inside that. There's no async work spanning multiple app-alive/app-dead transitions, no need to survive a process death mid-operation — it starts and finishes within one broadcast dispatch.

## Where this app draws the line, and why it doesn't need more

Compare this to what genuinely *would* require [[WorkManager]] or a [[Foreground services]]: if `syncGithubFolder` (a multi-second network-plus-database operation — see [[REST APIs]], [[Downloads]]) were triggered from a widget tap instead of from inside the running Flutter app, doing that synchronously in a `BroadcastReceiver` would be wrong — it would exceed the receiver's execution budget and risk being killed mid-sync. As it stands, this app deliberately keeps all its actual network/import work inside `NestedTreeNotifier`, triggered from UI the user has open and can watch a loading indicator for (`folder_screen.dart`'s `_syncGithubFolder` shows a `CircularProgressIndicator` dialog for exactly this reason) — never from a background trigger.

## Key takeaways

- Not all "runs when the app isn't foreground" code needs heavyweight background-execution APIs — a receiver doing one small, synchronous local database write (as this app's widget interactions do) is legitimately fine as-is.
- The dividing line is duration and reliability requirements: fast and synchronous (fits in a receiver's execution budget) vs. slow/network-bound/must-survive-process-death (needs [[WorkManager]] or a service).
- This app deliberately keeps its one genuinely slow operation (GitHub import/sync) inside the foreground Flutter UI rather than triggering it from any background entry point — sidestepping the need for background-execution APIs entirely rather than building around their constraints.
