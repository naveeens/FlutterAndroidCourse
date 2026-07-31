# Debugging widgets

The two workhorse tools are the Flutter Inspector (visual tree explorer, in your IDE or DevTools) and `debugPrint`/breakpoints for logic bugs. The Inspector is worth defaulting to first for *layout* bugs (why is this widget the wrong size, why is this overflowing) since it lets you click a widget on screen and see its exact constraints and render box, rather than guessing from code.

## Debugging patterns actually present in the nested app

**Selecting a widget on screen and reading its `key`.** `lib/folder_screen.dart`'s list gives every `NodeTile` a `ValueKey(nodeIdStr)`. If a reorder ever mis-assigns state to the wrong row, the fastest way to confirm it is to check with the Inspector that the `Element` under a given `NodeTile` carries the `ValueKey` you expect — that immediately tells you whether it's a keying bug (see [[Element tree]]) versus a data bug in `NestedTreeNotifier`.

**`debugPrint`-style state tracing for `ChangeNotifier` bugs.** Because `NestedTreeNotifier` (`lib/providers/tree_notifier.dart`) centralizes all tree mutations, a wrong count or a stale list after an operation is usually easiest to debug by adding a temporary print right after `notifyListeners()` in the relevant method — e.g., in `reorderNode`:

```dart
final newOrderIndex = (prevIndex + nextIndex) ~/ 2;
final updated = node.copyWith(orderIndex: newOrderIndex);
_nodes[newIndex] = updated;
notifyListeners();
// debugPrint('reordered to index $newIndex, orderIndex=$newOrderIndex');
await _nodeDao.update(updated);
```

This isolates whether a bug is in in-memory state (`_nodes`) or in what actually landed in SQLite — a distinction that matters a lot given `reorderNode` updates `_nodes` optimistically *before* awaiting the DB write.

**`error` state surfaced in the UI itself.** `_buildBody` in `folder_screen.dart` renders `notifier.error` directly when `_load()` catches an exception:

```dart
if (notifier.error != null) {
  return Center(child: Column(children: [
    Icon(Icons.error_outline_rounded, ...),
    Text(notifier.error!, ...),
  ]));
}
```

This is a debugging aid built into production code — rather than a silent failure or an unhandled exception crashing the widget, the actual `e.toString()` from the failed load is visible on screen, which is often enough to diagnose a DB or query bug without attaching a debugger at all.

**Async/`mounted` bugs** are common enough that `folder_screen.dart` checks `context.mounted` after nearly every `await` before touching `context` again (see [[BuildContext]]) — if you ever see "Looking up a deactivated widget's ancestor is unsafe" in the console, that's the exception this pattern exists to prevent, and the fix is almost always adding the missing `mounted` check.

## Key takeaways

- Use the Inspector for layout/sizing bugs (constraint mismatches, unexpected overflow) — it shows real `RenderObject` sizes, which is faster than reasoning about it from source.
- For state bugs in a `ChangeNotifier`, trace around `notifyListeners()` calls first — that's where "what changed" and "did the UI hear about it" meet.
- Surfacing caught errors directly in the UI (like `notifier.error`) during development turns a silent failure into an immediately visible one.
- "Looking up a deactivated widget's ancestor" almost always means a missing `mounted`/`context.mounted` check after an `await`.
