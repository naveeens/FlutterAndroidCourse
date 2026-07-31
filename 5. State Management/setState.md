# setState

`setState(VoidCallback fn)` is the most basic way a `State` object tells Flutter "something I hold has changed — call `build()` again." It runs `fn` synchronously (to actually mutate your fields) and then schedules a rebuild of that `State`'s widget. It's the right tool for state that's genuinely local to one widget and doesn't need to be seen or changed by anything else — the moment two unrelated widgets need to share state, that's the signal to lift it into a [[ChangeNotifier]]/[[Provider]] instead.

## In the nested app

`_FolderScreenState` (`lib/folder_screen.dart`) uses `setState` for pure UI-mode toggles that have no reason to live anywhere but this one screen — selection mode, which item is being inline-edited, whether the add bar is open:

```dart
void _handleLongPress(TreeNode node) {
  setState(() {
    if (!_selectionMode) {
      _selectionMode = true;
      _selectedIds.add('${node.type.name}_${node.id}');
    } else {
      if (_selectedIds.contains('${node.type.name}_${node.id}')) {
        _selectedIds.remove('${node.type.name}_${node.id}');
      } else {
        _selectedIds.add('${node.type.name}_${node.id}');
      }
    }
  });
}
```

Contrast this with `_deleteNode`, right next to it in the same file, which delegates to `notifier.deleteFolder(id)` / `notifier.deleteTask(id)` instead of `setState` — because deleting a node is data that other widgets (the folder-count badges, the search index) also need to know about, so it belongs in `NestedTreeNotifier` and travels via `notifyListeners()`, not `setState`.

`lib/widgets/add_item_bar.dart`'s `_cycleColor` is another clean local-only example:

```dart
void _cycleColor() {
  final priorities = List<int>.generate(AppColors.palette.length, (i) => i);
  final currentIndex = priorities.indexOf(_priority);
  final nextIndex = (currentIndex + 1) % priorities.length;
  setState(() => _priority = priorities[nextIndex]);
}
```

`_priority` only matters while the bar is open and being edited — nothing outside this widget needs to observe it until `_submit()` eventually hands it off to the notifier.

## The rule for choosing setState vs. lifting state up

Ask: does any widget *outside* this `State` object need to know when this value changes? If no — `setState` is correct and simplest. If yes — as with anything in `NestedTreeNotifier` (task counts, folder contents, breadcrumbs, all read by multiple independent widgets like `NodeTile`, `AppBarBreadcrumb`, and `FolderScreen` itself) — it needs to live in a shared `ChangeNotifier` instead.

## Key takeaways

- Mutate fields *inside* the `setState` callback, not before it — mutating outside and calling `setState(() {})` with an empty callback works but obscures which fields actually changed.
- Only call `setState` for fields `build()` actually reads — calling it for irrelevant fields wastes a rebuild; forgetting to call it for a field `build()` does read means the UI silently goes stale.
- `setState` after the `State` is disposed throws — this is exactly why async callbacks in this app guard with `if (!mounted) return;` before calling it (see [[Widget lifecycle]]).
- If you ever find two sibling `State` objects needing the same `setState`-managed value, that's the trigger to promote it to a `ChangeNotifier` instead of trying to pass callbacks between them.
