# Element tree

Flutter actually keeps three parallel trees, not one:

1. **Widget tree** — the immutable configuration you write in `build()` methods. Rebuilt (cheaply) often.
2. **Element tree** — the mutable, long-lived structure that mirrors the widget tree. Each `Element` holds a reference to the widget that currently configures it, and (for stateful widgets) to the `State` object. This is what actually persists across rebuilds.
3. **Render tree** — see [[Render tree]]; handles layout and painting.

When a parent rebuilds and produces a new widget for a given slot, Flutter doesn't throw away the old `Element` and build a fresh one — it *updates* the existing `Element` in place (`Element.update`) if the new widget has the same `runtimeType` and `key` as the old one. This is why a `State` object survives rebuilds: the `Element` that owns it stays put; only the widget it's configured by gets swapped out underneath it.

## Why this matters in the nested app

`lib/folder_screen.dart` uses `ValueKey` on each list item precisely because of how the element tree reconciles list children:

```dart
return NodeTile(
  key: ValueKey(nodeIdStr),
  node: node,
  ...
);
```

Without a stable key, `ReorderableListView` (backed by a `List<Element>`) matches old elements to new widgets *positionally* — element at index 3 gets updated with whatever widget is now at index 3. If the underlying data reorders (which is the entire point of a reorderable list — `notifier.reorderNode`), position-based matching would hand each `NodeTile`'s `Element` the *wrong* node's data, and worse, since `NodeTile` is a `StatefulWidget`, any per-tile `State` (like `AddItemBarState`'s in-progress edit) would stick to the wrong row.

`ValueKey(nodeIdStr)` — where `nodeIdStr` is `'${node.type.name}_${node.id}'` — tells the element tree "this element belongs to *this* node, wherever it ends up in the list." Reordering then correctly moves the `Element` (and its `State`) along with its data instead of reassigning state by position.

## Key takeaways

- The element tree is the layer that actually persists — it's why a `StatefulWidget`'s state survives even though a brand-new widget object is created on every `build()`.
- Keys tell the reconciliation algorithm "match by identity, not position" — essential for reorderable/filterable lists, exactly as `NodeTile`'s `ValueKey` demonstrates.
- An `Element` only gets discarded and rebuilt from scratch when the widget's `runtimeType` changes at that tree position (e.g., swapping a `Text` for an `Icon` in the same slot) — that's when you'd see a `State` reset unexpectedly.
