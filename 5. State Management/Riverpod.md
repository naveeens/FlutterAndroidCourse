# Riverpod

Riverpod is, roughly, "Provider's spiritual successor" from the same author — it solves the same problem (make values available to the widget tree, rebuild on change) but as a compile-time-safe, `BuildContext`-independent system of global `Provider` objects, rather than relying on the widget tree and `InheritedWidget` lookups the way [[Provider]] does. Its `Notifier`/`AsyncNotifier` classes are conceptually similar to a [[ChangeNotifier]], but state is replaced (not mutated), and Riverpod can catch a lot of "provider not found" mistakes at compile time that plain `Provider` only catches at runtime.

## Not used in the nested app

This app's `pubspec.yaml` depends on `provider`, not `flutter_riverpod`/`riverpod`. `NestedTreeNotifier` plus `ChangeNotifierProvider` (see [[Provider]] and [[ChangeNotifier]]) already covers this app's actual need — one shared piece of state, one screen tree, no complex dependency graph between multiple independent providers — which is exactly the situation where plain `Provider` is simplest and Riverpod's extra structure wouldn't pay for itself yet.

## What the same setup would look like in Riverpod

For comparison, roughly how `main.dart` and `folder_screen.dart` would change if this app used Riverpod instead:

```dart
// A NotifierProvider replaces ChangeNotifierProvider + the class declaration
final treeNotifierProvider = NotifierProvider<TreeNotifier, TreeState>(TreeNotifier.new);

class TreeNotifier extends Notifier<TreeState> {
  @override
  TreeState build() {
    // roughly what loadRoot() does today, run once on first read
    return const TreeState(nodes: [], loading: true);
  }

  Future<void> addFolder(String name, {int? priority}) async {
    final dao = ref.read(nodeDaoProvider);
    final id = await dao.insert(TreeNode(...));
    state = state.copyWith(nodes: [...state.nodes, ...]);
  }
}

// main.dart
void main() => runApp(const ProviderScope(child: NestedApp()));

// folder_screen.dart — needs a ConsumerWidget/ConsumerStatefulWidget instead of StatefulWidget
class FolderScreen extends ConsumerStatefulWidget { ... }
// reading: ref.watch(treeNotifierProvider) instead of context.watch<NestedTreeNotifier>()
// acting:  ref.read(treeNotifierProvider.notifier).addFolder(...) instead of context.read<NestedTreeNotifier>().addFolder(...)
```

The meaningful differences: providers become top-level `final` objects instead of something wired up in `main.dart`'s widget tree, `ref` replaces `context` as the thing you read providers through (which is what lets Riverpod code run outside the widget tree — in services, tests, background isolates), and `ref.watch` inside a `build()` auto-subscribes with automatic disposal when a widget stops watching, comparable to what `Consumer`/`context.watch` give you with `Provider` today.

## Key takeaways

- Riverpod ≈ Provider's ideas (shared, observable state) decoupled from `BuildContext` and made compile-time-checked — a real upgrade path if this app's dependency graph between shared state objects grew more complex.
- `ref.watch`/`ref.read` are Riverpod's analogues of `context.watch`/`context.read` — same rebuild-vs-one-off-read distinction discussed in [[Provider]] applies.
- For an app this size with one central `ChangeNotifier`, plain `Provider` (what's actually used here) is the simpler, sufficient choice — Riverpod's extra power (multiple independent, composable providers, testability outside widgets) matters more as an app's state graph grows.
