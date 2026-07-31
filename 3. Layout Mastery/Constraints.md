# Constraints

Flutter's layout protocol in one sentence: **constraints go down, sizes go up, parent sets position.** A parent hands each child a `BoxConstraints` (a min/max width and min/max height); the child picks any size within that range and reports it back; the parent then decides where to place that child. A child can never choose a size outside what its parent allowed, no matter what size it "wants."

This is why you can't just set a `width` on a widget and expect it to always take effect — if the parent's constraints are *tight* (min == max, e.g. inside a `SizedBox` or a `Row`'s cross axis under certain conditions), the child's own size preference is overridden entirely.

## In the nested app

`lib/widgets/add_item_bar.dart`'s bottom sheet content shows constraints being deliberately bounded so a growing `TextField` can't push the sheet off-screen:

```dart
ConstrainedBox(
  constraints: BoxConstraints(
    maxHeight: MediaQuery.of(context).size.height * 0.8,
  ),
  child: SingleChildScrollView(
    child: Column(
      mainAxisSize: MainAxisSize.min,
      children: [ TextField(maxLines: 3, minLines: 1, ...) ],
    ),
  ),
)
```

Without the `ConstrainedBox`, a multi-line `TextField` inside a `showModalBottomSheet` (which itself doesn't impose a max height by default) could in principle grow unbounded — capping it at 80% of screen height, then wrapping in `SingleChildScrollView`, is how the sheet stays usable regardless of how much text is typed.

`lib/folder_screen.dart`'s `_moveSelected` sheet does the same thing structurally with `DraggableScrollableSheet(initialChildSize: 0.6, maxChildSize: 0.9, ...)` — again, deliberately bounding what could otherwise be unconstrained content.

The classic *constraint violation* to watch for: nesting a `ListView` (which wants infinite height along its scroll axis) directly inside a `Column` with no `Expanded`. `folder_screen.dart` avoids this correctly — its `ReorderableListView.builder` is the direct child of `Scaffold.body`'s `SafeArea`, which hands it a bounded height from the screen itself, not from a `Column` that would otherwise offer infinite height.

## Key takeaways

- "Constraints go down, sizes go up, parent sets position" explains almost every layout bug you'll hit — when something overflows or throws a `RenderFlex`/unbounded-height error, trace what constraints its parent is actually handing it.
- `ConstrainedBox` (or a `Container` with `constraints:`) is how you *impose* a min/max where the ambient constraints would otherwise be too loose or too tight — as seen capping the add-item sheet at 80% of screen height.
- A scrollable widget (`ListView`, `SingleChildScrollView`) wants unbounded extent along its scroll axis — giving it one directly from a `Column`/`Row` without `Expanded`/a bounded parent is the single most common Flutter layout exception.
