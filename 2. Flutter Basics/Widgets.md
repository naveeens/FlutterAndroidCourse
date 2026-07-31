# Widgets

Everything on screen in Flutter is a widget — not just buttons and text, but padding, alignment, and even the app itself. A widget is an immutable description of a piece of UI: "at this point in the tree, render this, configured like this." Flutter takes your whole tree of widget descriptions and turns it into pixels every frame.

"Immutable" is the key word. A widget never mutates itself to reflect a state change — instead, when state changes, Flutter asks you to build a *new* tree of widget objects (cheap, since they're just lightweight configuration), and diffs it against the old one to figure out what actually needs to repaint. This is why `build()` methods should be fast and free of side effects: they might run often, and always produce a fresh description rather than edit an old one.

## In the nested app

`lib/main.dart` shows the whole-app widget nested inside configuration widgets:

```dart
runApp(
  ChangeNotifierProvider(
    create: (_) => NestedTreeNotifier()..loadRoot(),
    child: NestedApp(
      initialThemeKey: initialTheme,
      initialAccent: initialAccent,
    ),
  ),
);
```

`ChangeNotifierProvider`, `NestedApp`, and everything inside `NestedApp.build()` are all widgets — some carry actual pixels (`Icon`, `Text`), some carry layout instructions (`Padding`, `Row`), and some carry no UI at all, just data plumbing (`ChangeNotifierProvider`, `Consumer`). They compose by nesting: a widget's `child` (or `children`) is itself built from more widgets.

`lib/folder_screen.dart` is a good example of how deep and declarative this nesting gets — `Scaffold` → `SafeArea` → `Consumer` → `GestureDetector` → your actual list. Each layer adds one concern (screen chrome, safe-area insets, listening to the notifier, dismissing the keyboard on tap) without the others needing to know about it.

## Two flavors

Almost every widget you write is one of two kinds:

- **`StatelessWidget`** — described fully by its constructor arguments; see [[StatelessWidget]].
- **`StatefulWidget`** — holds mutable state across rebuilds via a companion `State` object; see [[StatefulWidget]].

## Key takeaways

- A widget is a lightweight, immutable *configuration*, not the on-screen element itself (that distinction is what [[Element tree]] and [[Render tree]] are for).
- Rebuilding is not automatically expensive — creating new widget objects is cheap, so don't be afraid of `build()` running often.
- Composition over configuration: instead of one widget with fifty parameters, Flutter nests many small single-purpose widgets. `EmptyState` in `lib/widgets/empty_state.dart` is a tiny, focused example of this.
