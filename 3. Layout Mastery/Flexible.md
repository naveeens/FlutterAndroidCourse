# Flexible

`Flexible` is `Expanded`'s more permissive sibling — both only work directly inside a `Row`/`Column`/`Flex`, both take a `flex` factor for sharing space with other flexible children, but they differ in `fit`:

- `Expanded` = `Flexible(fit: FlexFit.tight)` — the child is forced to fill its entire allotted share of space.
- `Flexible(fit: FlexFit.loose)` (the default for bare `Flexible`) — the child may be *up to* its allotted share, but can be smaller if its own content doesn't need that much.

In other words: `Expanded` says "take exactly this much room," `Flexible` says "take at most this much room."

## Why the nested app reaches for Expanded, not Flexible

Every flex-space case in this codebase — the title `Text` in `lib/widgets/node_tile.dart`, the `TextField` in `note_editor_screen.dart`'s editor, the horizontal-scrolling toolbar list — genuinely wants to *fill* the remaining space, not just cap itself at a maximum:

```dart
Row(
  children: [
    Icon(...),
    const SizedBox(width: 10),
    Expanded(child: Text(node.name, ...)), // should fill remaining width, always
    ...
  ],
)
```

If this had been `Flexible` instead of `Expanded`, a short folder name like "Work" would only occupy the width its text needs, and the checkbox/count badge after it would shift left to sit right next to the short text — visually broken for a list where every row needs consistent trailing-icon alignment. `Expanded` is what guarantees the text column is always exactly "remaining space," regardless of how short the name is.

## When you would want Flexible instead

`Flexible` is the right call when a child *can* shrink to fit its content but you still want it capped so it can't overflow past its siblings — e.g., a `Row` with an icon and a `Text` label where the label should truncate if the row gets tight, but shouldn't be artificially stretched wider than its text when there's extra room. If this app had, say, a tag-chip row where chips should hug their own label text but never overflow the screen, `Flexible` per chip (each with its own `Text(overflow: TextOverflow.ellipsis)`) would be the appropriate choice — `Expanded` would force every chip to stretch, which isn't the intent.

## Key takeaways

- `Expanded` = `Flexible(fit: FlexFit.tight)` — reach for the explicit `Expanded` name when you want to fill space (which is most of the time, as seen throughout `node_tile.dart` and `note_editor_screen.dart`).
- Use `Flexible(fit: FlexFit.loose)` when a child should be allowed to shrink-wrap its own content but must not overflow — think truncatable labels next to fixed icons.
- Mixing `Expanded` and `Flexible` siblings in the same `Row`/`Column` is valid — their `flex` values are compared against each other regardless of `fit`.
