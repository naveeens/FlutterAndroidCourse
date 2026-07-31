# ChangeNotifier

`ChangeNotifier` is a small mixin/base class that gives you an `addListener`/`removeListener` subscription mechanism plus one method, `notifyListeners()`, which calls every registered listener. That's the entire mechanism — no magic, no code generation. It's the foundation both `Provider`'s `ChangeNotifierProvider` and (indirectly) `ValueNotifier` are built on.

The pattern it enables: put your mutable app state and the logic that mutates it in one class, call `notifyListeners()` at the end of every method that changes something visible, and let widgets that care subscribe to it (typically via [[Provider]]).

## In the nested app

`NestedTreeNotifier` (`lib/providers/tree_notifier.dart`) is the app's single source of truth for the currently-viewed folder tree — it *is* a `ChangeNotifier`:

```dart
class NestedTreeNotifier extends ChangeNotifier {
  final NodeDao _nodeDao;

  int? _currentFolderId;
  int? get currentFolderId => _currentFolderId;

  List<TreeNode> _nodes = [];
  List<TreeNode> get nodes => List.unmodifiable(_nodes);
  ...
}
```

Every mutating method follows the same shape: change private state, then `notifyListeners()`. `addFolder` is representative:

```dart
Future<void> addFolder(String name, {int? priority}) async {
  ...
  final id = await _nodeDao.insert(folder);
  final newFolderNode = folder.copyWith(id: id);
  _nodes = [..._nodes, newFolderNode];
  notifyListeners();
  unawaited(WidgetBridge.refreshWidget());
}
```

Note the order: `notifyListeners()` fires as soon as in-memory state (`_nodes`) is updated, *before* the fire-and-forget `WidgetBridge.refreshWidget()` call — the UI doesn't wait on that. `_load()` calls `notifyListeners()` twice — once right after setting `_loading = true` (so the UI can immediately show a spinner) and once in the `finally` block after the data arrives (so the UI can show the result or an error). That's a common `ChangeNotifier` pattern: notify at every state transition a widget might care about, not just at the end.

Exposed getters return `List.unmodifiable(_nodes)` rather than the raw list — listeners can read the current state but can't mutate it directly and sneak state changes past `notifyListeners()`.

## Key takeaways

- `notifyListeners()` is the entire contract — call it any time observable state changes, and nowhere else (calling it when nothing actually changed just wastes a rebuild).
- Exposing unmodifiable views (`List.unmodifiable`, or just getters with no setters) keeps all mutation funneled through methods that remember to call `notifyListeners()`.
- `ChangeNotifier` itself has no opinion about *how* widgets subscribe — that's [[Provider]]'s job (`ChangeNotifierProvider` + `Consumer`/`context.watch`).
- `ChangeNotifier.dispose()` exists and matters if you ever create one manually outside a `Provider` (which manages disposal for you) — see [[Widget lifecycle]].
