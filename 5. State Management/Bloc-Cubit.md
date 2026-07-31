# Bloc / Cubit

Bloc (and its simpler sibling, Cubit) is a state management pattern/package (`flutter_bloc`) built around emitting immutable state objects rather than mutating an observable object in place. A `Cubit<State>` exposes methods that call `emit(newState)`; a full `Bloc<Event, State>` goes further and separates *events in* from *states out* explicitly, processing events through a stream. Widgets subscribe via `BlocBuilder`/`BlocListener`, similar in spirit to `Consumer` in [[Provider]].

The core philosophical difference from `ChangeNotifier`: Bloc states are typically immutable value objects (`copyWith`-style, often with `equatable` for value equality) that get *replaced* wholesale, versus a `ChangeNotifier` whose fields are mutated in place and which then announces "something changed" without necessarily saying what.

## Not used in the nested app

This app doesn't depend on `flutter_bloc` — its `pubspec.yaml` only lists `provider`. `NestedTreeNotifier` (see [[ChangeNotifier]]) plays the same architectural role a `Cubit<TreeState>` would in a Bloc-based version of this app: it's the single owner of tree data, and it's the thing UI subscribes to for updates.

## What the same logic would look like as a Cubit

For comparison, `NestedTreeNotifier.addFolder` — currently mutating `_nodes` in place and calling `notifyListeners()` — would become something like:

```dart
class TreeState {
  final List<TreeNode> nodes;
  final bool loading;
  final String? error;
  const TreeState({required this.nodes, this.loading = false, this.error});

  TreeState copyWith({List<TreeNode>? nodes, bool? loading, String? error}) =>
      TreeState(nodes: nodes ?? this.nodes, loading: loading ?? this.loading, error: error);
}

class TreeCubit extends Cubit<TreeState> {
  final NodeDao _nodeDao;
  TreeCubit(this._nodeDao) : super(const TreeState(nodes: []));

  Future<void> addFolder(String name, {int? priority}) async {
    final folder = TreeNode(type: NodeType.folder, name: name, ...);
    final id = await _nodeDao.insert(folder);
    emit(state.copyWith(nodes: [...state.nodes, folder.copyWith(id: id)]));
  }
}
```

The functional difference: `NestedTreeNotifier.nodes` can be read at any time as whatever it currently holds, with no history; a `Cubit`'s `emit` calls form a discrete sequence of `TreeState` snapshots, which is what makes Bloc's `bloc_test` package able to assert an exact sequence of expected states after an action — a stronger and more explicit test contract than asserting on a `ChangeNotifier`'s final field values.

## Key takeaways

- Bloc/Cubit trades `ChangeNotifier`'s mutate-in-place simplicity for immutable, replaceable state snapshots — better testability and traceability, more ceremony (state classes, `copyWith`, often `equatable`).
- If you added Bloc to this app, `NestedTreeNotifier`'s public methods map almost 1:1 onto `Cubit` methods; the DAOs it depends on wouldn't change at all — see [[Dependency Injection]].
- Neither is "correct" in the abstract — `ChangeNotifier`/`Provider` is lighter-weight and is what this codebase actually uses; Bloc pays off more on larger teams/apps that want stricter, more testable state transition contracts.
