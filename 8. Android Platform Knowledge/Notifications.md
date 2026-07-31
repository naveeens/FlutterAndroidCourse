# Notifications

Android notifications are the standard way an app surfaces information outside its own UI — in the status bar and notification shade — for things the user should know about even when the app isn't open: a completed download, an incoming message, a reminder.

## Not used in the nested app

This app posts zero notifications. It has no push messaging, no due-date reminders (see [[AlarmManager]] for the related absence of any time-scheduled triggers), and no long-running background operations that would need to report completion or progress via the status bar — its longest operation (GitHub import/sync) runs and completes entirely within the foreground UI, shown via an in-app `CircularProgressIndicator` dialog rather than a notification (see [[Foreground services]] for why that operation doesn't even need to survive backgrounding, let alone report status while backgrounded).

## Where notifications would fit if this app grew

The clearest hypothetical is the same one that comes up for [[AlarmManager]] and [[WorkManager]]: task due dates. "Remind me about this task at 5pm" would need, at minimum, a notification channel (mandatory setup, covered separately in [[Notification channels]]) and code roughly like:

```kotlin
val notification = NotificationCompat.Builder(context, CHANNEL_ID)
    .setContentTitle(task.name)
    .setContentText("Due now")
    .setSmallIcon(R.drawable.ic_notification)
    .setContentIntent(pendingIntentToOpenTask(task.id))
    .setAutoCancel(true)
    .build()
NotificationManagerCompat.from(context).notify(task.id.toInt(), notification)
```

The `setContentIntent` there would reuse exactly the pattern this app already has for opening a specific folder from outside the app — `NestedWidgetProvider.kt`'s `openAppIntent` (see [[PendingIntent]] and [[Intents]]) already demonstrates "build a `PendingIntent` that opens `MainActivity` with an extra telling it which folder/task to navigate to"; a notification tap would follow the identical shape, just fired from `NotificationManagerCompat.notify` instead of a widget tap.

A second, less obvious use case: GitHub sync completion. If `syncGithubFolder` were ever moved to run in the background (via [[WorkManager]], as discussed there) rather than always foreground-triggered, a completion/failure notification would be the natural way to report back to a user who's no longer watching the screen — mirroring how a file-download or backup app tells you "done" after you've moved on to something else.

## Key takeaways

- This app's complete lack of notifications is consistent with everything else about its design covered in this section — no scheduled/background work exists yet that would need to report status outside the foreground UI.
- Any notification-posting feature this app added would reuse its existing deep-link pattern (`PendingIntent` opening `MainActivity` with a folder/task id extra) for the tap action, not invent a new one.
- Due-date reminders are the single feature absence that would pull in `AlarmManager`, notifications, and notification channels together, as a connected trio — currently none of the three exist in this codebase.
