# Provider

`package:provider` is a thin, idiomatic wrapper around [[InheritedWidget]] for making an object available to a whole subtree without threading it through every constructor. `ChangeNotifierProvider` specifically pairs that with [[ChangeNotifier]]: it creates the notifier, hands it to descendants, listens for `notifyListeners()`, and disposes the notifier automatically when the provider itself is removed from the tree.

Three ways to consume a provided value, each with a different rebuild story:

- **`context.watch<T>()`** — subscribes; the calling widget rebuilds every time `T` calls `notifyListeners()`.
- **`Consumer<T>`** — same subscription, but scoped to just the `builder` region instead of the whole widget's `build()`, so you can watch without rebuilding unrelated sibling widgets.
- **`context.read<T>()`** — grabs the current value once, does *not* subscribe. Safe (and correct) inside callbacks like `onPressed`, wrong inside `build()` if you actually need to react to changes.

## In the nested app

`lib/main.dart` sets up the provider once, at the root, wrapping the whole app:

```dart
runApp(
  ChangeNotifierProvider(
    create: (_) => NestedTreeNotifier()..loadRoot(),
    child: NestedApp(...),
  ),
);
```

`lib/folder_screen.dart`'s body uses `Consumer` for the part of the tree that actually needs to redraw when the folder contents change:

```dart
body: SafeArea(
  child: Consumer<NestedTreeNotifier>(
    builder: (context, notifier, _) {
      return GestureDetector(
        onTap: () => FocusManager.instance.primaryFocus?.unfocus(),
        child: _buildBody(context, notifier),
      );
    },
  ),
),
```

Everywhere else in the same file, button handlers use `context.read<NestedTreeNotifier>()` instead — they're one-off actions (add a folder, delete a node, navigate) that shouldn't cause that call site itself to rebuild:

```dart
onPressed: () => context.read<NestedTreeNotifier>().togglePinForSelected(selectedNodes),
```

`lib/widgets/breadcrumb.dart` shows `Consumer` used for a small, self-contained subtree — only the breadcrumb bar rebuilds when `notifier.breadcrumbs` changes, not the entire app bar around it:

```dart
return Consumer<NestedTreeNotifier>(
  builder: (context, notifier, _) {
    final breadcrumbs = notifier.breadcrumbs;
    ...
  },
);
```

This granularity is deliberate performance hygiene: if `folder_screen.dart` used `context.watch` at the top of its whole `build()` instead of scoping the subscription with `Consumer`, *every* rebuild-triggering change to the notifier would re-run the entire screen's build method (app bar, popup menus, everything), not just the list.

## Key takeaways

- `context.read` for actions (inside callbacks); `context.watch`/`Consumer` for anything that should visually update when the data changes.
- Scope your `Consumer`/`context.watch` as narrowly as possible — wrapping just the widget that needs to rebuild (as `breadcrumb.dart` does) avoids rebuilding unrelated UI.
- `ChangeNotifierProvider.create` + `..loadRoot()` (in `main.dart`) is a common pattern: create the notifier and immediately kick off its initial async load in the same expression, using Dart's cascade operator.
- Provider disposes the `ChangeNotifier` it created automatically — you never call `.dispose()` on `NestedTreeNotifier` yourself.
