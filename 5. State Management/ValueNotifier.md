# ValueNotifier

`ValueNotifier<T>` is a `ChangeNotifier` specialized to hold exactly one value — set `.value = newValue` and it calls `notifyListeners()` for you automatically (only if the new value actually differs from the old one, via `==`). It's the right tool when what you're modeling is genuinely just "one piece of data that changes over time," rather than a whole object with multiple fields and methods like [[ChangeNotifier]] is suited for.

`ValueListenableBuilder<T>` is its companion widget — give it a `ValueNotifier<T>` and a `builder`, and only that builder region rebuilds when the value changes, without needing `setState` or a full `Provider` setup.

## In the nested app

The app doesn't declare its own `ValueNotifier`, but it uses one directly from the Flutter SDK: `TextField`'s `undoController` in `lib/widgets/note_editor_screen.dart` is an `UndoHistoryController`, which *is* a `ValueNotifier<UndoHistoryValue>` under the hood:

```dart
final _undoController = UndoHistoryController();
```

Rather than `ValueListenableBuilder` specifically, the screen uses the more general `ListenableBuilder` (works with anything `Listenable`, including plain `ValueNotifier`) to rebuild just the undo/redo buttons when the controller's value changes:

```dart
ListenableBuilder(
  listenable: _undoController,
  builder: (context, _) {
    final can = _undoController.value.canUndo;
    return IconButton(
      icon: Icon(Icons.undo_rounded, size: 20,
        color: can ? colors.onSurface.withValues(alpha: 0.7) : colors.onSurface.withValues(alpha: 0.28),
      ),
      tooltip: 'Undo',
      onPressed: can ? _undoController.undo : null,
    );
  },
)
```

This is exactly the pattern `ValueListenableBuilder` exists for — scoped rebuilds driven by one small piece of external, mutable state — just using the SDK's more general `ListenableBuilder` since `UndoHistoryController` already conforms to `Listenable`. Every keystroke in the `TextField` updates the controller's value; only these two `IconButton`s rebuild in response, not the whole editor screen.

## When you'd reach for it yourself

If this app needed, say, a live character counter under the note editor, a hand-rolled `ValueNotifier<int>` updated on every `TextField` change plus a `ValueListenableBuilder<int>` around just the counter `Text` would be the lightweight alternative to routing that through `setState` on the whole screen or promoting it into `NestedTreeNotifier`.

## Key takeaways

- `ValueNotifier<T>` = `ChangeNotifier` for a single value, with automatic `notifyListeners()` on `.value =` assignment.
- `ValueListenableBuilder`/`ListenableBuilder` scope a rebuild to exactly the widget that needs it — the app's undo/redo buttons are real, working proof of that scoping.
- Reach for `ValueNotifier` over a full `ChangeNotifier` class when there's genuinely just one value to track; reach for `ChangeNotifier` (as `NestedTreeNotifier` does) once you have multiple related fields and methods that mutate them together.
