# Row

`Row` lays its children out horizontally. It has two axes with different rules: the **main axis** (horizontal) is where `mainAxisAlignment` and `mainAxisSize` operate, and the **cross axis** (vertical) is where `crossAxisAlignment` operates. By default a `Row` tries to be as wide as its parent allows and sizes its height to its tallest child.

The rule that trips people up first: an unconstrained child inside a `Row` (like a `Text` that could be arbitrarily long) can overflow — a `Row` doesn't shrink its children to fit, it just lays them out at their natural size along the main axis. That's what [[Expanded]] and [[Flexible]] exist to fix.

## In the nested app

`lib/widgets/node_tile.dart`'s title row is a textbook `Row`: a fixed-size icon, a fixed-size pin indicator, an `Expanded` title that absorbs the remaining space, and a fixed-size trailing checkbox/counter:

```dart
Row(
  children: [
    ReorderableDragStartListener(index: widget.index, child: Icon(...)),
    if (isFolder && node.isPinned) ...[
      const SizedBox(width: 4),
      Icon(Icons.push_pin_rounded, size: 12, color: accent),
    ],
    const SizedBox(width: 10),
    Expanded(
      child: Text(node.name, style: theme.textTheme.bodyLarge?.copyWith(...)),
    ),
    if (isFolder) Consumer<NestedTreeNotifier>(builder: (context, notifier, _) { ... }),
    if (node.isTask) Transform.scale(scale: 1.2, child: Checkbox(...)),
  ],
)
```

Without the `Expanded` around the title `Text`, a long folder name would push the trailing checkbox off-screen or overflow — `Expanded` forces the text to take only what's left after its siblings claim their fixed widths, and `Text`'s own `overflow`/wrapping behavior kicks in from there.

`lib/folder_screen.dart`'s `_moveSelected` bottom sheet header nests `Row` differently — no `Expanded` needed because every child (`Container` circle, `SizedBox`, `Text`) has an intrinsic size and the `Row` is allowed to just be as wide as its content within the sheet.

## Key takeaways

- `Row` does not wrap or shrink children by default — an overflowing `Row` almost always means a child needs `Expanded`/`Flexible`, or the content itself needs `Text.overflow`/`softWrap` handling.
- Mix fixed-size children (icons, `SizedBox` spacers) with exactly one or a few `Expanded`/`Flexible` children to get "fixed elements plus one that fills remaining space" — the most common real-world `Row` shape, as seen in `NodeTile`.
- `mainAxisSize: MainAxisSize.min` makes a `Row` shrink-wrap its content instead of filling available width — useful inside something like a `Row` nested in a bottom sheet where you don't want it to stretch full-width.
