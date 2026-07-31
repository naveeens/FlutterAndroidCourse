# Drift ORM

Drift is a type-safe ORM/query builder layered on top of `sqflite` (or other SQLite bindings) — you write your schema as Dart classes (or `.drift` SQL files with codegen), and Drift generates typed table classes, typed query methods, and reactive `Stream`-based queries (a query result stream that auto-updates when underlying rows change), instead of you writing raw SQL strings and manually mapping `Map<String, dynamic>` rows to objects.

## Not used in the nested app — this app writes sqflite by hand instead

`lib/db/node_dao.dart` and `lib/db/note_dao.dart` are exactly the layer Drift would generate for you. Compare the manual version actually in this app:

```dart
Future<TreeNode?> getById(int id) async {
  final db = await _db;
  final rows = await db.query('nodes', where: 'id = ?', whereArgs: [id]);
  if (rows.isEmpty) return null;
  return TreeNode.fromMap(rows.first);
}
```

against roughly what the Drift equivalent would look like:

```dart
class Nodes extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get type => text()();
  TextColumn get name => text()();
  IntColumn get parentId => integer().nullable().references(Nodes, #id)();
  ...
}

// Generated, typed, and Drift catches a typo'd column name at compile time:
Future<Node?> getById(int id) => (select(nodes)..where((n) => n.id.equals(id))).getSingleOrNull();
```

The manual version has a real, present risk Drift removes: `TreeNode.fromMap(rows.first)` (in `lib/models/tree_node.dart`) hand-maps string column keys (`'parent_id'`, `'order_index'`) to fields — a typo in one of those strings is a runtime failure (or silent `null`), not a compile error. Drift's generated column accessors make that class of bug impossible.

The recursive task-count query is the sharpest illustration of the actual trade-off. This app's version is raw SQL via `rawQuery`, because it needs a `WITH RECURSIVE` CTE:

```dart
final rows = await db.rawQuery('''
  WITH RECURSIVE tree(root_id, id) AS ( ... ) SELECT ... FROM tree JOIN nodes ...
''', [actualParentId]);
```

Drift *does* support raw/custom SQL escape hatches for exactly this kind of query (`customSelect`), so this specific query wouldn't actually get simpler under Drift — but every other query in `node_dao.dart`/`note_dao.dart` (the `getChildren`, `insert`, `update`, `softDelete` methods) would gain compile-time column/type checking and Drift's reactive `Stream` queries, which could replace some of `NestedTreeNotifier`'s manual `notifyListeners()`-after-write bookkeeping (see [[ChangeNotifier]]) with queries that update themselves automatically when the underlying table changes.

## Key takeaways

- Drift's core value is compile-time-checked schema and queries plus generated boilerplate — trading codegen setup for eliminating the exact class of stringly-typed bug `TreeNode.fromMap` is exposed to today.
- It still sits on top of real SQLite, so nothing about this app's relational, tree-shaped data model would need to change to adopt it — only the DAO layer (`node_dao.dart`, `note_dao.dart`) would.
- Drift's reactive streaming queries are a genuine architectural alternative to this app's `ChangeNotifier`-plus-manual-`notifyListeners()` pattern — worth knowing about even though this app took the simpler, more manual path.
- For a schema this size (two tables), hand-written `sqflite` DAOs are a completely reasonable choice — Drift's benefit compounds more on larger schemas with many tables and queries, where hand-mapped `Map<String, dynamic>` rows become a real maintenance risk.
