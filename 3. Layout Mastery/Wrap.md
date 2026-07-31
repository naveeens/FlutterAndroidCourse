# Wrap

`Wrap` behaves like `Row`/`Column` but, instead of overflowing when children don't fit on one line, it wraps them onto additional "runs" (rows, if `direction: Axis.horizontal`, or columns if vertical) — the layout equivalent of CSS `flex-wrap`. It's the right tool for a set of same-ish-sized items (chips, tags, filter pills) whose total count doesn't fit a fixed-width row and shouldn't scroll, just reflow.

## Why the nested app doesn't use it

The nested app doesn't currently have a variable-count-of-small-items UI that needs wrapping — its closest candidates for "many small items in a row" are the priority/color picker in `lib/widgets/add_item_bar.dart` (cycled one at a time via `_cycleColor`, tap-to-advance, not laid out as a row of swatches) and the formatting toolbar in `note_editor_screen.dart`, which is deliberately a horizontally-*scrolling* `ListView` rather than a wrapping row:

```dart
Expanded(
  child: ListView.builder(
    scrollDirection: Axis.horizontal,
    itemCount: actions.length,
    itemBuilder: (context, index) => ...,
  ),
)
```

That's a legitimate alternative design decision worth noticing: scrolling keeps the toolbar at a fixed single-line height regardless of how many formatting actions exist, while `Wrap` would let it grow to two or three lines and eat into the editor's vertical space. If this toolbar ever needed to show *all* actions without scrolling, swapping the `ListView` for a `Wrap` would be the natural change — but it would also mean giving up the fixed 48px toolbar height.

## What it would look like here

If the color picker in `add_item_bar.dart` were redesigned as a visible row of swatches instead of a tap-to-cycle dot, `Wrap` is exactly the widget for it:

```dart
Wrap(
  spacing: 8,
  runSpacing: 8,
  children: AppColors.palette.asMap().entries.map((entry) {
    final selected = entry.key == _priority;
    return GestureDetector(
      onTap: () => setState(() => _priority = entry.key),
      child: CircleAvatar(radius: 12, backgroundColor: entry.value),
    );
  }).toList(),
)
```

`spacing` controls the gap between items in the same run; `runSpacing` controls the gap between runs. Neither exists on `Row`/`Column`, since they don't have the concept of multiple runs.

## Key takeaways

- `Wrap` is for reflowing many similar-sized items across lines; `Row`/`Column` don't wrap at all (they overflow), and a horizontally-scrolling `ListView` (as this app's toolbar uses) is the alternative when you want to *keep* a single line rather than grow vertically.
- `spacing`/`runSpacing` replace the manual `SizedBox` spacers you'd otherwise thread between every child in a `Row`.
- Choosing `Wrap` vs. a scrolling list is a real design decision, not just a technical one — `Wrap` grows the container to fit content; a scrolling list keeps the container's size fixed and hides overflow behind a scroll gesture.
