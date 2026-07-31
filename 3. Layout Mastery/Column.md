# Column

`Column` is `Row`'s vertical twin: children stack top-to-bottom, the main axis is vertical (`mainAxisAlignment`, `mainAxisSize`), the cross axis is horizontal (`crossAxisAlignment`). By default it sizes its height to the sum of its children and its width to its widest child, up to whatever its parent allows.

The most common `Column` bug is putting an unbounded child (like a `ListView`) directly inside one without wrapping it in `Expanded` — a `Column` gives unconstrained height to children that ask for it, and a scrollable child asking for infinite height throws a layout error. The fix is always the same: wrap the flexible child in `Expanded` so it's told "take what's left," not "take whatever you want."

## In the nested app

`lib/widgets/note_editor_screen.dart`'s `_buildEditor` shows exactly this pattern — a `TextField` that should fill all available vertical space, above a fixed-height toolbar:

```dart
Widget _buildEditor(ThemeData theme, ColorScheme colors) {
  return Column(
    children: [
      Expanded(
        child: Padding(
          padding: const EdgeInsets.fromLTRB(16, 8, 16, 8),
          child: TextField(
            controller: _ctrl,
            maxLines: null,
            expands: true,
            ...
          ),
        ),
      ),
      _buildFormattingToolbar(colors, theme),
    ],
  );
}
```

Without `Expanded`, `expands: true` on the `TextField` (which tells *it* to fill its parent) would have nothing bounded to fill, and the layout would fail. The pattern reads naturally once you know the rule: "the flexible thing in the middle gets `Expanded`; the fixed things around it (here, the toolbar) don't."

`bin_screen.dart`'s `AlertDialog` content is the opposite case — a `Column` isn't even needed there since `content` is a single `Text`, but its sibling widgets (`title`, `actions`) show how `Column`-based layouts compose naturally when a dialog needs multiple stacked elements.

## Key takeaways

- A `Column` sizes to its children's natural height unless told otherwise — an unbounded/scrollable child needs `Expanded` to get a concrete height to work with.
- `crossAxisAlignment: CrossAxisAlignment.start` is one of the most-reached-for `Column` properties, since the default (`center`) surprises people used to left-aligned text stacks.
- Same overflow logic as [[Row]], just on the vertical axis — if a `Column` overflows vertically, look for a child that needs `Expanded`/`Flexible`, or wrap the whole `Column` in a scrollable.
