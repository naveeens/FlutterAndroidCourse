# Responsive layouts

"Responsive" covers two related but distinct problems: adapting to *different screen sizes* (phone vs. tablet vs. desktop — usually via `LayoutBuilder` or breakpoints on `MediaQuery.size`), and adapting to *changing insets on the same screen* (keyboard appearing, system nav bar, notches — via `MediaQuery.viewInsets`/`padding`). The nested app is a phone-only app, so it doesn't do the first kind, but it does the second kind extensively and correctly, which is worth studying since it's the more universally-needed skill.

## In the nested app

**Keyboard-aware sheets.** `lib/folder_screen.dart`'s accent-picker and JSON-import bottom sheets both pad their bottom edge by the keyboard's current height:

```dart
padding: EdgeInsets.fromLTRB(16, 20, 16, MediaQuery.of(ctx).viewInsets.bottom + 24),
```

`viewInsets.bottom` is 0 when the keyboard is hidden and grows to the keyboard's height when it's shown — this single line is what keeps the sheet's buttons above the keyboard instead of hidden behind it, on every device and keyboard height, without hardcoding anything.

**Safe-area-aware positioning.** The same file's inline add bar:

```dart
bottomSheet: _isAddingInline
    ? Padding(
        padding: EdgeInsets.only(bottom: MediaQuery.of(context).padding.bottom),
        child: AddItemBar(key: _addBarKey, isInline: true, ...),
      )
    : null,
```

`MediaQuery.of(context).padding.bottom` is the height of the system gesture bar/nav bar inset — devices with a 3-button nav bar, a gesture pill, or neither all get correctly-placed content because the padding is read at runtime, not assumed.

**Conditional bottom padding based on keyboard visibility**, in `note_editor_screen.dart`'s formatting toolbar:

```dart
final hasKeyboard = MediaQuery.of(context).viewInsets.bottom > 0;
return SafeArea(
  top: false,
  bottom: !hasKeyboard,
  child: Container(height: 48, ...),
);
```

Here the toolbar's own `SafeArea` bottom inset is deliberately *disabled* when the keyboard is showing — because in that state the keyboard itself is what's occupying the bottom of the screen, and adding safe-area padding on top of it would create an unwanted gap.

## Key takeaways

- `MediaQuery.of(context).viewInsets.bottom` is the keyboard height right now — the standard way to keep interactive content above the keyboard.
- `MediaQuery.of(context).padding` is safe-area insets (status bar, notch, gesture nav) — `SafeArea` is usually the higher-level way to consume it, but reading it directly (as `folder_screen.dart` does) is appropriate when you need to combine it with other padding.
- True multi-screen-size responsiveness (tablet/desktop layouts, breakpoints) is a separate concern this app doesn't need — see [[Adaptive layouts]] for that side of the topic.
