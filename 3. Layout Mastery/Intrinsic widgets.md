# Intrinsic widgets

`IntrinsicHeight` and `IntrinsicWidth` force their child (and, transitively, its subtree) to be measured at its "natural" size along one axis *before* normal layout happens, then apply that size as a constraint. The classic use case: a `Row` of children with different natural heights (say, a `Text` and a taller custom widget) where you want every child stretched to match the *tallest* one — `Row`'s `crossAxisAlignment: CrossAxisAlignment.stretch` alone can't do this because it stretches to the `Row`'s own height, which is unbounded/parent-derived, not "as tall as my tallest sibling."

The important caveat, and the reason to reach for these sparingly: intrinsic sizing requires an extra full layout pass to measure the "natural" size before the real layout pass runs, so it's measurably more expensive than normal layout. It doesn't work at all with some widgets (like most scrollables), and Flutter will assert if you try.

## Why the nested app doesn't use it

Nothing in this codebase needs "size every sibling to match the tallest one." The closest structural need — dividers matching content height, like the vertical divider between the formatting toolbar and the mic button in `note_editor_screen.dart` — is solved with an explicit fixed height instead:

```dart
Widget _verticalDivider(ColorScheme colors) => Container(
  width: 1,
  height: 24,
  color: colors.onSurface.withValues(alpha: 0.1),
);
```

Because the toolbar's `Container` already has a known, fixed `height: 48`, there's no ambiguity to resolve at layout time — the divider is just given an explicit height that looks right (24, roughly matching the icon size) rather than needing to be computed from a sibling's intrinsic size. This is the more common real-world outcome: most "I need matching heights" problems are solved more cheaply with an explicit fixed size once you know what you're aiming for, and `IntrinsicHeight` is reserved for cases where the matching size genuinely isn't known ahead of time (e.g., a `Row` of `Card`s with variable, data-driven content whose heights should all match the tallest one).

## Key takeaways

- `IntrinsicHeight`/`IntrinsicWidth` make siblings match a computed "natural" size — the right tool only when that size genuinely can't be known/fixed ahead of time.
- They cost an extra layout pass; avoid them in performance-sensitive lists (e.g., don't wrap a `ListView` item builder's row in `IntrinsicHeight` if a fixed height or `CrossAxisAlignment.stretch` would do).
- If you can hardcode or otherwise derive a fixed size (as the app's toolbar divider does), that's almost always preferable to intrinsic sizing.
