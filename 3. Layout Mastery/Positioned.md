# Positioned

`Positioned` only means something inside a [[Stack]] — it tells the stack "place this child at these exact offsets from my edges" instead of letting the stack's `alignment` decide. You supply any combination of `top`/`bottom`/`left`/`right`/`width`/`height`; over-constraining (e.g. `left`, `right`, and `width` all at once) is an error, same as over-constraining any box in Flutter's layout model — see [[Constraints]].

## In the nested app

The hint bubble in `lib/widgets/note_editor_screen.dart`'s `VoiceRecordButton` (see [[Stack]] for the full context) is the app's one clear `Positioned` usage:

```dart
Stack(
  alignment: Alignment.center,
  clipBehavior: Clip.none,
  children: [
    if (_showHint)
      Positioned(
        bottom: 50,
        right: 0,
        child: AnimatedOpacity(...),
      ),
    AnimatedContainer(...), // the mic icon itself, unpositioned
  ],
)
```

`bottom: 50, right: 0` places the hint bubble's bottom-right corner 50px above the stack's bottom edge and flush with its right edge — that's what puts the "Tap to speak" bubble floating just above and to the side of the mic icon, rather than overlapping it directly. Only `bottom` and `right` are given, so the bubble's `width`/`height` come from its own content (the padded `Text`), and its `top`/`left` are left unconstrained — it simply extends upward and leftward from the pinned corner.

## Key takeaways

- `Positioned` is meaningless outside a `Stack` — if you find yourself wanting `Positioned`-like behavior somewhere else, you actually want a `Stack` wrapping the relevant widgets first.
- Give only the edges you actually need pinned (as `VoiceRecordButton` does with just `bottom`/`right`) — the unspecified edges and size are then derived from the child's own content, which is usually what you want for something like a tooltip or badge.
- Positioning by edge offsets, not by explicit x/y coordinates, is deliberate — it keeps the positioned child correct if the `Stack` itself resizes (e.g. due to text scaling or a different screen size).
