# Notification channels

Since Android 8 (API 26), every notification must belong to a channel — a user-configurable category (importance level, sound, vibration, whether it shows on the lock screen) that the user controls per-channel in system settings, rather than an all-or-nothing per-app toggle. An app creates its channels once (typically at startup), and every individual notification it posts afterward must reference one.

## Not used in the nested app

Consistent with [[Notifications]] having no notification-posting code at all, there's no `NotificationChannel` creation anywhere in this app — no `NotificationManager.createNotificationChannel(...)` call, no channel id constant, nothing.

## What it would look like, and why it's not just boilerplate

If this app added task-due-date reminders (the recurring hypothetical across this section — see [[AlarmManager]], [[Notifications]]), channel setup would need to happen once, early (typically in `MainActivity.onCreate` or an `Application` subclass), before any reminder notification could be posted:

```kotlin
val channel = NotificationChannel(
    "task_reminders",
    "Task reminders",
    NotificationManager.IMPORTANCE_DEFAULT,
).apply {
    description = "Notifies you when a task's due time arrives"
}
val manager = getSystemService(NotificationManager::class.java)
manager.createNotificationChannel(channel)
```

The reason this matters beyond ceremony: if this app ever grew *two* meaningfully different kinds of notifications — say, due-date reminders and GitHub sync completion status — they'd want **separate channels**, specifically so a user who wants due-date alerts to make sound but doesn't care about sync-completion pings (or vice versa) can configure that themselves in system settings, without an in-app settings screen needing to replicate that control. `createNotificationChannel` is also idempotent/safe to call every launch with the same channel id — re-creating an existing channel with the same id is a no-op for anything the user has already customized, so it's normal to just call it unconditionally on every app start rather than checking "does this channel already exist."

## Key takeaways

- Required (Android 8+) before posting any notification — skipping channel creation means `notify()` either silently fails or throws, depending on API level.
- One channel per meaningfully distinct *category* of notification the user might want to control independently — not one channel per individual notification, and not necessarily just one channel for the whole app either.
- Safe to call `createNotificationChannel` unconditionally on every app start — it won't overwrite a user's existing per-channel settings for a channel id that already exists.
- This app has no notifications yet, so no channels — but any reminder/notification feature discussed elsewhere in this section (see [[Notifications]], [[AlarmManager]]) would need this as a prerequisite step.
