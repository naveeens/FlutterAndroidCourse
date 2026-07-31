# Align

`Align` positions its child within itself according to an `Alignment` (e.g. `Alignment.centerLeft`, `Alignment.topRight`), and — unlike [[Positioned]] — it works anywhere, not just inside a `Stack`. Given loose constraints, `Align` sizes itself to fit its child plus whatever the alignment implies; you can also force it to grow via `widthFactor`/`heightFactor`, or let its parent's constraints dictate its size.

`Center` is just `Align` with `alignment: Alignment.center` baked in — worth knowing so you recognize they're the same mechanism.

## In the nested app

`lib/widgets/note_editor_screen.dart`'s markdown checkbox builder uses `Align` to left-anchor a small fixed-size checkbox tile inside a padded region that would otherwise center it by default:

```dart
Padding(
  padding: const EdgeInsets.only(top: 3.5, right: 8.0),
  child: Align(
    alignment: Alignment.centerLeft,
    child: Container(
      width: 18,
      height: 18,
      decoration: BoxDecoration(
        color: isChecked ? colors.primary : Colors.transparent,
        borderRadius: BorderRadius.circular(5.0),
        border: isChecked ? null : Border.all(color: colors.onSurface.withValues(alpha: 0.45), width: 1.8),
      ),
      child: isChecked ? Icon(Icons.check_rounded, size: 13, color: colors.onPrimary) : null,
    ),
  ),
)
```

This checkbox sits inside a markdown-rendered line of text, where the surrounding layout could otherwise let it float anywhere within its allotted box — `Align(alignment: Alignment.centerLeft)` pins it to the start, matching where a real checkbox glyph would sit relative to the list-item text next to it.

`Center` (the more common relative in this codebase) shows up throughout for loading and empty states — `lib/folder_screen.dart`'s `_buildBody`:

```dart
if (notifier.loading) {
  return const Center(child: CircularProgressIndicator.adaptive());
}
```

and `lib/widgets/empty_state.dart`'s whole layout is `Center(child: Column(...))` — both are `Align` under the hood, just via its more common named form.

## Key takeaways

- `Align` (and its `Center` shorthand) works standalone, unlike `Positioned` which requires a `Stack` parent — reach for `Align` when you just need "put this child at one spot within its box," and `Stack`+`Positioned` when you need multiple overlapping children at different spots.
- `Alignment` values run from `-1.0` to `1.0` on each axis (`Alignment(-1, -1)` is top-left, `Alignment(1, 1)` is bottom-right) — named constants like `centerLeft`/`topRight` cover the common cases, but custom fractional alignment is available when you need it.
- An `Align` with loose parent constraints shrinks to its child's size (plus alignment factor) rather than filling available space — if you want it to fill space *and* align content within that space, its parent needs to hand it tight/bounded constraints (e.g. via `Expanded` or a sized parent).
