# SQLite

SQLite is a full relational database in a single file — no server process, just a `.db` file on disk. `package:sqflite` is Flutter's standard binding to it on Android/iOS. It's the right choice once your data has real structure and relationships (parent/child folders, tasks with notes) and you want to query it (filter, sort, join, aggregate) rather than just load-the-whole-blob, which is where key-value stores like [[SharedPreferences]] stop being enough.

## In the nested app

This app's entire tree of folders and tasks lives in one SQLite database. `lib/db/database_helper.dart` owns the schema and connection lifecycle:

```dart
class DatabaseHelper {
  static const _dbName = 'nested.db';
  static const _dbVersion = 9;
  static const int rootSentinelId = 0;

  Database? _db;
  Future<Database> get database async {
    _db ??= await _initDb();
    return _db!;
  }

  Future<Database> _initDb() async {
    final dbPath = await getDatabasesPath();
    final path = join(dbPath, _dbName);
    return openDatabase(path, version: _dbVersion, onCreate: _onCreate, onConfigure: _onConfigure, onUpgrade: _onUpgrade);
  }
}
```

`_db ??= await _initDb()` is a lazy singleton — the database only opens once, on first access, and every later call reuses the same connection. Two tables: `nodes` (folders and tasks are the *same* table, distinguished by a `type` column — see the schema in `_createNodeTable`) and `notes` (one-to-many against a task's `id` via `task_id`).

`lib/db/node_dao.dart` is where actual queries live — a DAO (Data Access Object) per table, keeping SQL out of the UI/business-logic layer. A plain `WHERE`-based query:

```dart
Future<List<TreeNode>> getChildren(int? parentId) async {
  final db = await _db;
  final actualParentId = parentId ?? DatabaseHelper.rootSentinelId;
  final rows = await db.query('nodes',
    where: 'parent_id = ? AND deleted_at IS NULL AND id != 0',
    whereArgs: [actualParentId],
    orderBy: 'order_index ASC',
  );
  return rows.map(TreeNode.fromMap).toList();
}
```

and a recursive `WITH RECURSIVE` common table expression (raw SQL, via `rawQuery`, for something the query builder can't express) that computes done/total task counts for an entire folder subtree in one round-trip:

```dart
final rows = await db.rawQuery('''
  WITH RECURSIVE tree(root_id, id) AS (
    SELECT id, id FROM nodes WHERE type = 'folder' AND deleted_at IS NULL AND parent_id = ?
    UNION ALL
    SELECT tree.root_id, nodes.id FROM nodes JOIN tree ON nodes.parent_id = tree.id WHERE nodes.deleted_at IS NULL
  )
  SELECT tree.root_id AS folder_id, SUM(...) AS total, SUM(...) AS done
  FROM tree JOIN nodes ON nodes.id = tree.id
  GROUP BY tree.root_id HAVING total > 0
''', [actualParentId]);
```

That query is a good illustration of *why* you'd reach for real SQL over a simpler store — computing recursive subtree aggregates like this in Dart, by walking the tree manually, would be both slower and far more code.

## Key takeaways

- `db.query(...)` for straightforward filtered/sorted reads; `db.rawQuery`/`db.execute` for anything the builder can't express (joins, CTEs, aggregates) — this app uses both, choosing raw SQL only where it's genuinely needed.
- Soft deletes (`deleted_at IS NOT NULL`, see `softDelete`/`getTrashed` in `node_dao.dart`) are a common SQLite pattern for a recycle-bin feature — the row stays, a timestamp marks it hidden, and a real `DELETE` only happens on `hardDelete`.
- `ON DELETE CASCADE` foreign keys (`parent_id INTEGER REFERENCES nodes(id) ON DELETE CASCADE`) push cascading-delete logic into the database itself — but only if `PRAGMA foreign_keys = ON` is set, which `_onConfigure` does explicitly (SQLite defaults this off).
- See [[Data migration]] for how this schema evolves across app versions via `_onUpgrade`.
