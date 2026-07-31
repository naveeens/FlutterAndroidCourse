# Expanded

`Expanded` tells a `Row`/`Column`/`Flex` "give this child a share of the remaining space along the main axis, after fixed-size siblings have taken theirs." It only works as a direct child of a `Row`, `Column`, or `Flex` — it's meaningless anywhere else, because it relies on the parent's flex layout algorithm to divide up leftover space.

Multiple `Expanded` children share space in proportion to their `flex` value (default `1`), so two `Expanded` widgets with no explicit `flex` split the remaining space 50/50.

## In the nested app

The clearest example is `lib/widgets/node_tile.dart`'s title row, discussed in [[Row]]:

```dart
Row(
  children: [
    Icon(...),               // fixed size
    const SizedBox(width: 10),
    Expanded(
      child: Text(node.name, ...),
    ),
    if (isFolder) Consumer<NestedTreeNotifier>(...), // fixed size
    if (node.isTask) Checkbox(...),                  // fixed size
  ],
)
```

The icon, spacer, count badge, and checkbox all claim exactly the space they need; `Expanded` is what makes the title `Text` absorb everything left over — and because `Text` handles overflow internally (ellipsis/wrap), a long folder name degrades gracefully instead of breaking the row's layout.

`note_editor_screen.dart`'s `_buildEditor` uses `Expanded` the "Column" way, discussed in [[Column]] — the `TextField` gets all vertical space not claimed by the fixed-height formatting toolbar below it:

```dart
Column(
  children: [
    Expanded(child: Padding(..., child: TextField(expands: true, ...))),
    _buildFormattingToolbar(colors, theme),
  ],
)
```

And the formatting toolbar itself nests another `Expanded` inside its own `Row`, so the horizontally-scrolling list of formatting buttons takes all width not claimed by the fixed-width vertical divider and mic button next to it:

```dart
Row(
  children: [
    Expanded(child: ListView.builder(scrollDirection: Axis.horizontal, ...)),
    _verticalDivider(colors),
    Padding(..., child: VoiceRecordButton(...)),
  ],
)
```

## Key takeaways

- `Expanded` only works directly inside `Row`/`Column`/`Flex` — wrapping it anywhere else (or wrapping a widget that's already in a non-flex parent) throws a "RenderFlex" error or does nothing.
- Exactly one `Expanded` among otherwise fixed-size siblings is the most common shape — "these things are fixed, this one thing fills the rest."
- Multiple `Expanded`s split space by their `flex` ratio; use unequal `flex` values (`flex: 2` vs `flex: 1`) to give one side more room than another without hardcoding pixel widths. See also [[Flexible]] for the non-forcing variant.
