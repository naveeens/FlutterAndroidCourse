# Widget lifecycle

A `State` object goes through a predictable sequence of callbacks: `initState()` once when it's first inserted into the tree, `build()` whenever it (or an ancestor) needs to redraw, `didUpdateWidget()` when the parent rebuilds it with new configuration, and `dispose()` once when it's permanently removed. Getting this sequence right is mostly about two things: doing one-time setup in `initState` (not `build`, which can run many times) and always releasing anything you acquired, in `dispose`.

## In the nested app

`lib/folder_screen.dart`'s `_FolderScreenState` is a clean example of the full pattern — acquire in `initState`, release in `dispose`:

```dart
@override
void initState() {
  super.initState();
  // Hot-start: the widget header was tapped while the app is already running.
  _widgetNavSub = WidgetNavService.instance.openFolderStream.listen(_handleOpenFolder);
  // Cold-start: the app was launched by a widget tap.
  WidgetsBinding.instance.addPostFrameCallback((_) async {
    final initialFolderId = await WidgetNavService.instance.consumeInitialFolderId();
    if (initialFolderId != null) _handleOpenFolder(initialFolderId);
  });
}

@override
void dispose() {
  _widgetNavSub?.cancel();
  _scrollController.dispose();
  super.dispose();
}
```

Two lifecycle details worth noticing here:

1. **`addPostFrameCallback` in `initState`.** You can't safely trigger navigation or show dialogs *during* `initState` — the widget tree isn't finished building yet. Scheduling the callback for after the first frame is the standard workaround for "I need to do something UI-ish right after this widget mounts."
2. **The `mounted` check.** `_handleOpenFolder` starts with `if (!mounted) return;`. Async work started in `initState` can finish *after* the widget has already been disposed (user navigated away while a `Future` was in flight) — calling `setState` on a disposed `State` throws. `mounted` is how you guard against that.

`lib/widgets/add_item_bar.dart`'s `AddItemBarState` shows the same acquire/release discipline for a `TextEditingController` and `FocusNode`:

```dart
final _titleController = TextEditingController();
final _focusNode = FocusNode();

@override
void initState() {
  super.initState();
  ...
  _focusNode.addListener(_onFocusChange);
}

@override
void dispose() {
  _titleController.dispose();
  _focusNode.dispose();
  super.dispose();
}
```

Forgetting `dispose()` here wouldn't crash immediately, but it leaks — the controller and focus node (and their listeners) stay alive in memory after the widget is gone.

## Key takeaways

- `initState` runs once per `State` object; `build` can run many times — never do one-time setup (subscribing to a stream, creating a controller) inside `build`.
- Anything with a `.dispose()` method that you created yourself needs you to call it in `dispose()`. Framework-managed objects you didn't construct (like an inherited `Theme`) are not your responsibility.
- Guard any `setState` inside an async callback with `if (!mounted) return;` first — see `_handleOpenFolder` above, and the same pattern after every `await` in `folder_screen.dart`'s dialogs (`if (!context.mounted) return;`).
- `dispose()` cannot be async — cancel subscriptions and dispose controllers synchronously, always calling `super.dispose()` last.
