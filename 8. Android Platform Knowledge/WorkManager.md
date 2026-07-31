# WorkManager

`WorkManager` is Android's recommended API for deferrable, guaranteed background work — tasks that should run even if the app is killed or the device reboots, optionally constrained (only on WiFi, only while charging) and optionally periodic. It's the modern replacement for older, less reliable scheduling mechanisms, and the right default whenever work needs to *eventually* complete but doesn't need to happen at an exact instant.

## Not used in the nested app — and a look at the closest candidate feature

Nothing in this app schedules deferred or periodic background work. Its one multi-step, potentially-slow operation — `syncGithubFolder` in `lib/providers/tree_notifier.dart` (download a repo's markdown files, reconcile against local SQLite; see [[REST APIs]]) — is always triggered directly by the user tapping "Sync with GitHub" in `folder_screen.dart`, runs while that screen is open, and shows a blocking progress dialog until it finishes:

```dart
Future<void> _syncGithubFolder(BuildContext context) async {
  final notifier = context.read<NestedTreeNotifier>();
  final folderId = notifier.currentFolderId;
  if (folderId == null) return;

  showDialog(context: context, barrierDismissible: false,
      builder: (_) => const Center(child: CircularProgressIndicator()));

  try {
    await notifier.syncGithubFolder(folderId);
    ...
  } catch (e) { ... }
}
```

This is a legitimate, simpler alternative to `WorkManager` for this app's actual need: the sync is short enough (a handful of seconds for a typical markdown repo) that making the user wait with visible progress is a fine UX, and there's no requirement that it survive the app being killed or complete "eventually" in the background — if the user closes the app mid-sync, there's no promise being broken, since they initiated and were watching the operation.

## What would push this app toward WorkManager

If GitHub sync became automatic — say, "keep my imported folders in sync every few hours, even if I never open the app" — that's exactly the shape of problem `WorkManager` solves and ad-hoc foreground-triggered `Future`s don't:

```kotlin
val syncRequest = PeriodicWorkRequestBuilder<GithubSyncWorker>(6, TimeUnit.HOURS)
    .setConstraints(Constraints.Builder().setRequiredNetworkType(NetworkType.CONNECTED).build())
    .build()
WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "github_sync", ExistingPeriodicWorkPolicy.KEEP, syncRequest,
)
```

Note this would be Kotlin/native-side scheduling calling into a `Worker` — the actual sync *logic* (`NestedTreeNotifier.syncGithubFolder` equivalent) would either need a Dart-callable background entry point (Flutter's `workmanager` plugin bridges this) or a parallel native implementation talking to the same SQLite file the way `NodeDbHelper.kt` already does for the widget.

## Key takeaways

- `WorkManager` is for work that must eventually run and survive process death/reboot, optionally on a schedule or under constraints — not simply "any operation that takes more than an instant."
- A foreground-triggered `Future` with a visible progress indicator (this app's actual `syncGithubFolder` flow) is the right, simpler choice whenever the operation is short and the user is present and waiting for it — reaching for `WorkManager` here would be unnecessary complexity.
- The trigger to reconsider: the moment a task needs to run *without* the user having just asked for it in the foreground (a periodic auto-sync, a deferred retry after the app was killed), that's when `WorkManager` earns its complexity.
