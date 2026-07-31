# Hive

Hive is a pure-Dart, NoSQL key-value/object database — no native SQL engine underneath, just fast binary serialization to disk. Its appeal is speed and a very low-ceremony API (`box.put(key, value)`, `box.get(key)`) for storing Dart objects directly (via generated `TypeAdapter`s), without writing SQL or a schema at all.

## Not used in the nested app

This app depends on `sqflite`, not `hive`/`hive_flutter`. The data model here — folders and tasks in a **tree**, with parent/child relationships, ordering, recursive task-count aggregation across subtrees, and free-text search across both node names and note content — is exactly the shape of problem a relational engine like SQLite is built for: `NodeDao.getRecursiveTaskCounts` (see [[SQLite]]) runs a recursive CTE joining across the tree in one query, something a key-value store like Hive has no native way to express — you'd have to load relevant objects into Dart and walk/aggregate them by hand.

## Where Hive would have been a reasonable choice instead

If this app's data were flatter — say, if "notes" had no relationship to a tree of folders, and were just a list of independent objects keyed by id, with no cross-note queries needed — Hive's simplicity would be a fair trade against `sqflite`'s. Roughly, a Hive-based `NoteDao` might look like:

```dart
@HiveType(typeId: 1)
class Note {
  @HiveField(0) final int id;
  @HiveField(1) final int taskId;
  @HiveField(2) final String content;
  Note({required this.id, required this.taskId, required this.content});
}

class NoteBox {
  static late Box<Note> _box;
  static Future<void> init() async => _box = await Hive.openBox<Note>('notes');

  Future<void> insert(Note note) => _box.put(note.id, note);
  Note? get(int id) => _box.get(id);
  List<Note> getByTask(int taskId) =>
      _box.values.where((n) => n.taskId == taskId).toList(); // linear scan — no index
}
```

That last line is the tell: `getByTask` becomes a full linear scan over every note in the box, because Hive has no query language or indexes — you filter in Dart after loading. `NoteDao.getByTask` in the real app instead runs `WHERE task_id = ?` against an indexed SQLite column (`idx_notes_task`), which stays fast regardless of how many notes exist. That gap — SQL query/index support vs. manual filtering — is the main reason this app didn't reach for Hive.

## Key takeaways

- Hive excels at simple, flat, per-key object storage with minimal setup — no SQL, no schema migrations to hand-write, very fast reads/writes.
- It has no query language or indexing — any "find all X matching Y" beyond a direct key lookup means loading and filtering in Dart, which doesn't scale the way an indexed SQL `WHERE` clause does.
- This app's actual needs — relational data (parent/child trees), recursive aggregation, and `LIKE`-based search across two tables (`NestedTreeNotifier.searchTree`) — are squarely in SQLite's territory, which is why `sqflite` was the right call here over Hive.
