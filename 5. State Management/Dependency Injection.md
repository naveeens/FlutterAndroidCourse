# Dependency Injection

Dependency injection just means: a class receives the things it depends on from outside (usually via its constructor) instead of constructing them itself internally. The payoff is testability and flexibility — a caller can hand in a fake/mock dependency instead of the real one, without the dependent class needing to know or care.

You don't need a DI *framework* to do this in Dart/Flutter — constructor parameters with sensible defaults get you most of the way, and that's exactly what this app does.

## In the nested app

`NestedTreeNotifier` (`lib/providers/tree_notifier.dart`) takes its `NodeDao` as an optional constructor parameter rather than hardcoding `NodeDao()` inside itself:

```dart
class NestedTreeNotifier extends ChangeNotifier {
  final NodeDao _nodeDao;
  ...
  NestedTreeNotifier({NodeDao? nodeDao}) : _nodeDao = nodeDao ?? NodeDao();
  ...
}
```

In normal app code (`lib/main.dart`), nobody passes `nodeDao` — the default (`NodeDao() `, the real SQLite-backed implementation) is used:

```dart
ChangeNotifierProvider(create: (_) => NestedTreeNotifier()..loadRoot(), ...)
```

But because the dependency is *injectable*, a test can construct `NestedTreeNotifier(nodeDao: FakeNodeDao())` with an in-memory fake instead — testing all the tree logic (breadcrumb management, reordering, import/export JSON handling) without touching a real database at all. `NestedTreeNotifier` itself never has to know whether it's talking to real SQLite or a test double; it only knows about the `NodeDao` interface it was handed.

This is the whole idea in miniature: `?? NodeDao()` gives you a sane default for production so callers don't have to think about it, while the parameter itself keeps the class decoupled from one specific concrete implementation. Every DAO in the app (`NodeDao`, `NoteDao`) is a plain class with no dependencies of its own, so this same pattern — constructor parameter, defaulted to the real implementation — is what you'd reach for anywhere else in the app that needed the same testability.

## Key takeaways

- "Dependency injection" in Dart is often just: accept a nullable/optional constructor parameter for a dependency, default it to the real implementation. No framework required, as `NestedTreeNotifier({NodeDao? nodeDao})` demonstrates.
- The benefit only materializes if the dependency is referenced through an interface/type the caller could substitute — hardcoding `NodeDao()` inline inside every method that needs it (instead of storing it as `_nodeDao` from the constructor) would defeat the purpose even if the parameter existed.
- This pattern and [[Unit tests]]/[[Mocking]] go together directly — injectability is what makes a fake/mock swap possible in the first place.
- Package-based DI (`get_it`, `riverpod`'s providers) solves a different problem — *locating* dependencies across a large app without threading them through many constructors — which this app's size doesn't yet need.
