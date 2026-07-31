# Isar Database

Isar is a NoSQL, object-oriented database for Flutter that — unlike [[Hive]] — *does* support indexes and a real query builder (`.filter()`, compound and full-text indexes), while still storing plain Dart objects rather than requiring hand-written SQL. It sits architecturally between Hive's raw simplicity and a full relational engine like [[SQLite]].

## Not used in the nested app

This app uses `sqflite` directly. Isar could technically express most of what this app needs — indexed lookups by `parent_id`, full-text-ish search over `name`/note content — but two things in the actual codebase point specifically at SQL rather than an object database:

1. **Recursive tree aggregation.** `NodeDao.getRecursiveTaskCounts` (see [[SQLite]]) uses a `WITH RECURSIVE` common table expression to compute done/total task counts across an arbitrarily deep folder subtree in a single database round-trip. Isar's query builder has no equivalent for recursive graph/tree traversal — you'd fetch and walk the tree in Dart yourself, which is more code and (for a deep tree) more round-trips.
2. **Cross-table `LIKE` search.** `NestedTreeNotifier.searchTree` runs one raw SQL query joining `nodes` and a subquery over `notes`:

   ```dart
   final rows = await db.rawQuery('''
     SELECT * FROM nodes 
     WHERE (name LIKE ? OR id IN (SELECT task_id FROM notes WHERE content LIKE ?))
       AND deleted_at IS NULL AND id != 0
     LIMIT 40
   ''', [rQuery, rQuery]);
   ```

   Isar does support its own indexed queries and even a `where().anyX()`-style API, but expressing "match against one table OR match against a subquery on a related table" as cleanly as raw SQL does here is exactly the kind of relational query that plays to SQL's strengths, not an object database's.

## What an Isar version would look like, roughly

For a flatter piece of this app's data — say, if notes were being modeled completely independently of the folder tree — an Isar collection is genuinely less code than the SQLite equivalent:

```dart
@collection
class Note {
  Id id = Isar.autoIncrement;
  @Index() late int taskId;
  late String content;
}

// Query: indexed, no SQL string to write
final notes = await isar.notes.filter().taskIdEqualTo(taskId).findAll();
```

No `CREATE TABLE`, no `TypeAdapter` boilerplate (code-gen handles it), and the `@Index()` annotation gives you the same lookup performance `idx_notes_task` gives `NoteDao.getByTask` in SQLite — for straightforward per-field queries like this one, Isar and SQLite would look about equally good. The gap only opens up once you need multi-table joins or recursive structure, which is exactly this app's situation.

## Key takeaways

- Isar improves meaningfully on Hive with real indexes and a query builder, while staying schema-light (annotate a class, no `CREATE TABLE`) — a strong middle ground for apps with per-collection, non-relational queries.
- It still lacks SQL's join/recursive-CTE expressiveness — this app's tree-shaped data and cross-table search are exactly the case where that gap shows up.
- Choosing between Isar and SQLite for a new project is really a question of how relational your data is: flat collections with indexed lookups favor Isar's lower ceremony; genuinely relational, tree-shaped, or join-heavy data (like this app's folders/tasks/notes) favors SQL.
