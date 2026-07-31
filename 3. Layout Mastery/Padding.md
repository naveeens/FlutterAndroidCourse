# Padding

`Padding` inserts empty space between its child and whatever bounds it — nothing more. It's one of the most common widgets in any Flutter app because layout in Flutter has no implicit spacing (unlike CSS box-model defaults); every gap is a widget you placed on purpose.

## In the nested app

`lib/folder_screen.dart` uses `Padding` for two very different jobs, both visible in the same file:

**Fixed spacing**, in the app bar's popup menu items:

```dart
PopupMenuItem(
  value: _MenuAction.theme,
  height: 40,
  padding: const EdgeInsets.symmetric(horizontal: 14),
  child: Row(children: [...]),
),
```

**Dynamic spacing that responds to system UI**, in the inline add bar's bottom sheet:

```dart
bottomSheet: _isAddingInline
    ? Padding(
        padding: EdgeInsets.only(bottom: MediaQuery.of(context).padding.bottom),
        child: AddItemBar(...),
      )
    : null,
```

That second one matters: `MediaQuery.of(context).padding.bottom` is the height of the system navigation bar/gesture inset, which varies by device. Hardcoding a padding value here would either leave a gap on devices without a gesture bar or clip the add bar behind the nav bar on devices with one — `Padding` driven by `MediaQuery` is what makes it correct on both.

`note_editor_screen.dart`'s JSON/GitHub import sheets go further, padding for the *keyboard*:

```dart
padding: EdgeInsets.fromLTRB(16, 20, 16, MediaQuery.of(ctx).viewInsets.bottom + 24),
```

`viewInsets.bottom` is how much of the screen the on-screen keyboard currently covers — adding it to the bottom padding keeps the sheet's content above the keyboard instead of hidden underneath it.

## Key takeaways

- `EdgeInsets.all`, `.symmetric`, `.only`, and `.fromLTRB` cover almost every case — pick the most specific one for what you're actually padding, it documents intent better than always reaching for `.all`.
- `MediaQuery`-driven padding (`padding.bottom` for safe areas, `viewInsets.bottom` for the keyboard) is how you make spacing correct across devices instead of guessing a fixed number — see [[Responsive layouts]].
- Padding around a `child` vs. `contentPadding` on the widget itself (like `TextField`'s `InputDecoration.contentPadding`) are different mechanisms — check whether the widget you're using already exposes its own padding property before wrapping it externally.
