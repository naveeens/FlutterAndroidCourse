# InheritedWidget

`InheritedWidget` is the low-level mechanism Flutter itself uses to make a value available to an entire subtree without passing it through every constructor — `Theme.of(context)`, `MediaQuery.of(context)`, and `Navigator.of(context)` are all backed by one. A descendant that calls `context.dependOnInheritedWidgetOfExactType<T>()` (which `.of(context)` wraps) both retrieves the current value *and* registers itself to rebuild whenever that `InheritedWidget` is replaced with a new instance whose `updateShouldNotify` returns `true`.

`package:provider`'s `ChangeNotifierProvider` is built directly on top of this mechanism (an `InheritedNotifier` internally) — see [[Provider]]. Understanding `InheritedWidget` is really understanding what `Provider` (and `Theme`, and `MediaQuery`) are doing for you under the hood.

## In the nested app

The app never defines a custom `InheritedWidget` — it uses `Provider`'s higher-level API for its own state (`NestedTreeNotifier`) and consumes Flutter's *built-in* `InheritedWidget`s constantly. `lib/folder_screen.dart`'s app bar alone hits three different ones in a few lines:

```dart
AppBar _buildAppBar(BuildContext context) {
  final theme = Theme.of(context);        // InheritedWidget: nearest Theme
  final colors = theme.colorScheme;
  final text = theme.textTheme;
  ...
}
```

and elsewhere in the same widget:

```dart
padding: EdgeInsets.only(bottom: MediaQuery.of(context).padding.bottom), // InheritedWidget: MediaQuery
...
Navigator.push(context, MaterialPageRoute(builder: (_) => const BinScreen())); // Navigator.of(context) internally
```

Every one of these calls walks up the element tree from `context`'s position looking for the nearest ancestor of that specific `InheritedWidget` subtype — which is also *why* `BuildContext` matters so much (see [[BuildContext]]): the same call made from a different widget's `context` could resolve to a different `Theme` if, say, that widget were wrapped in its own local `Theme` override, as `note_editor_screen.dart` does:

```dart
return Theme(
  data: theme, // a modified copy — primary color swapped to the note's accent
  child: PopScope(
    ...
    child: Scaffold(...),
  ),
);
```

Everything inside that `Theme`'s subtree now sees the accent-colored theme via `Theme.of(context)`, while everything outside it still sees the app's normal theme — that's `InheritedWidget` scoping working exactly as designed, and it's the same mechanism `ChangeNotifierProvider` would use if you nested a second, narrower provider partway down the tree instead of only at the app root.

## Key takeaways

- `.of(context)` (`Theme.of`, `Provider.of`, `MediaQuery.of`) is always "walk up from this context to the nearest ancestor `InheritedWidget` of this type" — it's why the *same* code can return different values depending on which widget's `context` calls it.
- Wrapping a subtree in a new `Theme` (or a nested `Provider`) scopes that value to just that subtree — descendants outside it are unaffected, exactly like `NoteEditorScreen`'s locally-overridden accent theme.
- You'll rarely write a raw `InheritedWidget` by hand in application code — `Provider` covers the common case (a value plus change notifications) with much less boilerplate.
