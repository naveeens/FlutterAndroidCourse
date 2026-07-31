# StatefulWidget

A `StatefulWidget` is actually two objects working together: the widget itself (immutable, just configuration — recreated every rebuild) and a `State` object (long-lived, survives across rebuilds, holds mutable fields). When you call `setState(() { ... })` inside the `State`, you're telling Flutter "some field I hold has changed — call `build()` again to reflect it."

The split exists because widgets are cheap and disposable but sometimes you need something that *isn't*: a `TextEditingController`, a `StreamSubscription`, a boolean flag that shouldn't reset every time a parent rebuilds this widget.

## In the nested app

`lib/folder_screen.dart`'s `_FolderScreenState` is a good full-featured example — it owns a `ScrollController`, a `Set<String>` of selected item ids, and several UI-mode flags (`_selectionMode`, `_isAddingInline`, `_editingNodeId`), none of which come from a parent:

```dart
class _FolderScreenState extends State<FolderScreen> {
  bool _selectionMode = false;
  final Set<String> _selectedIds = {};
  final ScrollController _scrollController = ScrollController();
  bool _isAddingInline = false;
  String? _editingNodeId;
  ...
}
```

Long-press on a node toggles selection mode by mutating that state and calling `setState`:

```dart
void _handleLongPress(TreeNode node) {
  setState(() {
    if (!_selectionMode) {
      _selectionMode = true;
      _selectedIds.add('${node.type.name}_${node.id}');
    } else {
      ...
    }
  });
}
```

Every field read inside `setState`'s callback (or read in `build()` afterward) is what Flutter diffs to decide what to repaint — `_selectionMode` flipping to `true` is what makes the app bar swap from breadcrumbs to a "N selected" title and selection-mode icons on the next `build()`.

`lib/widgets/add_item_bar.dart` shows the other classic reason to reach for `StatefulWidget`: it owns a `TextEditingController` and `FocusNode`, both of which need to live across rebuilds and be explicitly cleaned up — see [[Widget lifecycle]].

## Key takeaways

- Only call `setState` for fields that actually affect what `build()` renders — mutating a field without `setState` won't repaint anything; calling `setState` for something `build()` never reads is wasted work.
- `widget.someField` (from inside a `State`) is how you read the *current* widget configuration — it can change between rebuilds even though the `State` object itself doesn't.
- Every controller/subscription created in `initState` must be disposed in `dispose` — see [[Widget lifecycle]] for why the app crashes otherwise.
