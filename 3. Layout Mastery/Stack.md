# Stack

`Stack` lays children on top of one another instead of side-by-side or top-to-bottom. By default children are sized to their own natural size and aligned per `Stack.alignment`; a child wrapped in [[Positioned]] instead places itself at explicit offsets from the `Stack`'s edges. It's the tool for anything that needs to visually overlap: badges on icons, floating hints, custom-positioned overlays.

## In the nested app

`lib/widgets/note_editor_screen.dart`'s `VoiceRecordButton` is a self-contained `Stack` example — a floating "Tap to speak" hint that appears above the mic icon without affecting the icon's own layout:

```dart
Stack(
  alignment: Alignment.center,
  clipBehavior: Clip.none,
  children: [
    if (_showHint)
      Positioned(
        bottom: 50,
        right: 0,
        child: AnimatedOpacity(
          opacity: 1.0,
          duration: const Duration(milliseconds: 150),
          child: Container(
            padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 4),
            decoration: BoxDecoration(
              color: colors.surfaceContainerHighest.withValues(alpha: 0.95),
              borderRadius: BorderRadius.circular(20),
            ),
            child: Text(_state == RecordState.recording ? 'Tap to stop' : 'Tap to speak', ...),
          ),
        ),
      ),
    AnimatedContainer(
      duration: const Duration(milliseconds: 150),
      padding: const EdgeInsets.all(10),
      decoration: BoxDecoration(color: buttonColor, shape: BoxShape.circle),
      child: Icon(icon, color: iconColor, size: 22),
    ),
  ],
)
```

Two details worth noting: the mic icon (unpositioned, just centered by `Stack.alignment`) determines the `Stack`'s own size, and the hint bubble is `Positioned` 50px above and flush right — deliberately placed *outside* where the icon itself sits. `clipBehavior: Clip.none` is what allows that hint to render outside the `Stack`'s own bounding box instead of being clipped at its edge, which matters because the hint pops up above the icon's natural footprint.

The `if (_showHint)` conditional inside `children` is also a nice pattern: rather than always including the `Positioned` hint and toggling its opacity to zero, it's added to the tree only when needed — cheaper, and simpler to reason about than juggling visibility flags on a permanently-present widget.

## Key takeaways

- Unpositioned children in a `Stack` size and place themselves normally (subject to `alignment`); `Positioned` children opt out of that and place themselves at explicit edge offsets instead — see [[Positioned]].
- `clipBehavior: Clip.none` is needed whenever a positioned child is meant to render outside the stack's own bounds (tooltips, badges that poke past an edge) — the default clips them.
- A `Stack`'s size is determined by its non-positioned children (or the largest one, if there are several) — a `Stack` containing *only* `Positioned` children needs an explicit size from its parent, since it has nothing else to size itself by.
