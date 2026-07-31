# AlarmManager

`AlarmManager` schedules code to run at a specific future time (or repeating interval), waking the device if necessary even from deep sleep (`setExactAndAllowWhileIdle`, etc.). It's the tool for "this must happen at this wall-clock moment" — an alarm clock, a reminder at a specific time — as opposed to [[WorkManager]], which is for deferrable work that just needs to eventually run under given constraints, without caring about the exact instant.

## Not used in the nested app — this app has no time-based triggers at all

Nothing in this codebase schedules future execution. There are no due dates, no reminders, no recurring notifications — [[Notifications]] covers this from the other direction, but it's worth stating directly here too: the app's entire data model (`TreeNode` in `lib/models/tree_node.dart`, backed by `nodes`/`notes` tables in SQLite) has no timestamp field representing "when this should happen" or "remind me at," only `deleted_at` (for the recycle bin — see [[SQLite]]) and implicit ordering via `order_index`. There's simply no feature yet that has a reason to schedule anything for a future wall-clock time.

The app's home-screen widget explicitly opts out of even the *system's* time-based refresh mechanism, which is adjacent to this topic and worth noting:

```xml
<appwidget-provider ... android:updatePeriodMillis="0" ... />
```

`updatePeriodMillis="0"` disables Android's built-in periodic widget-refresh alarm entirely (the system would otherwise use `AlarmManager` internally to wake the widget provider on a schedule, typically no more often than every 30 minutes) — this app refreshes the widget explicitly and immediately whenever data actually changes (via `WidgetBridge.refreshWidget()`, called from Dart), which is both more timely and cheaper than any alarm-driven periodic poll could be.

## Where this app would need it

The obvious feature that would introduce a real `AlarmManager` (or, more likely today, its `WorkManager`-based `setExactAndAllowWhileIdle` equivalent, since `WorkManager` is generally preferred unless exact-time delivery is essential) is task due dates with reminders — "notify me about this task at 9am tomorrow." That would need, per task, scheduling an exact alarm at the due time, handling the alarm firing (likely via a `BroadcastReceiver`) by posting a notification (see [[Notifications]], [[Notification channels]]), and re-scheduling on device reboot (`AlarmManager` alarms don't survive a reboot by default — you'd listen for `BOOT_COMPLETED` and re-arm anything still pending).

## Key takeaways

- `AlarmManager` = exact wall-clock-time execution, including waking a sleeping device; `WorkManager` = deferrable, constraint-based execution without caring about the exact moment — pick based on whether the *exact time* genuinely matters.
- This app's widget deliberately turns off the system's own alarm-driven periodic refresh (`updatePeriodMillis="0"`) in favor of event-driven refresh triggered directly by data changes — a good example of "don't poll on a timer when you can be told exactly when something changed instead."
- Due dates/reminders are the natural feature that would require real `AlarmManager` usage in this app — currently absent, along with any other time-triggered behavior.
