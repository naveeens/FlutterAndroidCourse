# BuildContext

Every `build()` method receives a `BuildContext` — a handle to that widget's location in the tree. It's not configuration and it's not state; it's how a widget finds things above it (theme, inherited data, ancestors) and reaches services tied to that location (navigation, the nearest `Scaffold`). Almost every `Theme.of(context)`, `Navigator.of(context)`, or `context.read<T>()` call in the app works by walking up the tree from that widget's position looking for the nearest matching ancestor.

## In the nested app

`lib/folder_screen.dart` uses `context` for several distinct purposes in the same file, which is a good tour of what it's actually for:

```dart
// 1. Reading theme data from the nearest Theme ancestor
final theme = Theme.of(context);
final colors = theme.colorScheme;

// 2. Reading a Provider without subscribing to rebuilds (a one-off action)
final notifier = context.read<NestedTreeNotifier>();

// 3. Pushing a new route using the Navigator ancestor
Navigator.push(context, MaterialPageRoute(builder: (_) => const BinScreen()));
```

`context.read<T>()` vs. wrapping something in `Consumer<T>`/`context.watch<T>()` is a distinction worth internalizing: `read` grabs the current value once and does *not* rebuild this widget when it changes later; `watch`/`Consumer` subscribes, so the widget rebuilds every time. `folder_screen.dart`'s button handlers use `context.read` (they just want to call a method, not rebuild when the notifier's data changes), while its body uses `Consumer<NestedTreeNotifier>` (see [[Provider]]) because it needs to redraw whenever the tree data changes.

A subtler case is `context.mounted`, used throughout `folder_screen.dart` after every `await`:

```dart
final selectedNode = await showSearch<TreeNode?>(context: context, delegate: TreeSearchDelegate());
if (selectedNode != null && mounted) {
  await notifier.navigateToNodeFolder(selectedNode);
}
```

A `BuildContext` can become invalid while an `await` is in flight (the widget got removed from the tree). Using it afterward — say, calling `Navigator.of(context)` — throws. Checking `mounted` (on the `State`) or `context.mounted` first is required any time you use `context` after an `await`.

## Key takeaways

- `BuildContext` identifies *where* in the tree you are, which is what lets `.of(context)` lookups find the nearest matching ancestor (`Theme`, `Navigator`, a `Provider`).
- `context.read<T>()` for one-off reads/actions; `context.watch<T>()` or `Consumer<T>` for "rebuild me when this changes." Using `watch` where you meant `read` causes unnecessary rebuilds; using `read` where you meant `watch` means your UI goes stale.
- Never use a `BuildContext` after an `await` without checking `mounted`/`context.mounted` first — see every dialog handler in `folder_screen.dart` for the pattern.
- Each widget gets its *own* `BuildContext` tied to its own position — a `context` captured in one widget's `build()` is not interchangeable with a child's or parent's.
