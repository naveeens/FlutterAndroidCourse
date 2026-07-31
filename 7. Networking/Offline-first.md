# Offline-first

An offline-first app treats local storage as the source of truth and the network as an optional, secondary enhancement — the app is fully usable with no connection, and network operations sync *into* local state rather than local state depending on the network to function at all.

## The nested app is offline-first by construction

This isn't a network feature bolted onto a cloud-backed app — it's the reverse. Every core feature (creating folders/tasks, editing notes, reordering, search, the recycle bin) reads and writes exclusively through `lib/db/database_helper.dart`'s local SQLite database. `NestedTreeNotifier` (`lib/providers/tree_notifier.dart`) never touches the network for any of its core CRUD methods — `addFolder`, `addTask`, `toggleTaskCompletion`, `reorderNode`, `deleteFolder`, and so on all resolve entirely against `NodeDao`/`NoteDao`. Turn off the device's networking entirely and every one of those still works exactly the same.

Networking in this app is opt-in and additive, confined to two explicit user actions: **importing** from GitHub (`importFromGithub`) and **syncing** an already-imported folder (`syncGithubFolder`). Both write their results into the same local SQLite tables everything else uses — once an import completes, that data is now just local data like anything the user typed by hand, fully available offline from then on:

```dart
Future<void> importFromGithub(String repoUrl) async {
  ...
  final files = await GithubImportService.fetchMarkdownFiles(repoUrl);
  ...
  await _writeGithubFiles(wrapperId, files);
  await _load(_currentFolderId);
  ...
}
```

If that `fetchMarkdownFiles` call fails (no connection, repo not found), it throws — `folder_screen.dart`'s `_showGithubImportSheet` catches it and shows the error in the sheet itself, but nothing about the rest of the app is affected. Compare that to `syncGithubFolder`, which explicitly treats the *remote repo* as the source of truth only for the folders that opted into GitHub import — files no longer present remotely get soft-deleted locally, and new files get added — but again, this reconciliation only runs when the user explicitly asks for it, never automatically or as a precondition for using the app.

## What "true" offline-first with sync usually adds

A more elaborate offline-first system (think: a notes app that syncs across your own devices) typically adds a queue of pending local mutations that get replayed against a server once connectivity returns, plus conflict resolution for edits made on two devices while both were offline. This app doesn't need that machinery because it has no multi-device sync target at all — its only "remote" is a read-mostly external GitHub repo, reconciled on demand rather than continuously.

## Key takeaways

- Offline-first means local storage is authoritative and the network is optional, not the other way around — this app already has that property, just by virtue of every core feature being SQLite-backed with import/sync as clearly separated, optional operations.
- An import/sync failure should degrade to "that one operation didn't work" (as `_showGithubImportSheet`'s try/catch does), never "the app doesn't function" — a good litmus test for whether an app is really offline-first.
- If this app ever added real multi-device sync, that's the point it would need a mutation queue and conflict resolution — meaningfully more than what `importFromGithub`/`syncGithubFolder`'s one-way reconciliation does today.
