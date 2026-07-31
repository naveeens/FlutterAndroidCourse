# StatelessWidget

A `StatelessWidget` has no mutable state of its own. Everything it renders comes from its constructor arguments (its "configuration") plus whatever it reads from `context` (theme, inherited data). Given the same inputs, its `build()` always produces the same output. When it needs to change what's on screen, that has to come from *outside* — a parent rebuilding it with new arguments, or an `InheritedWidget`/`Provider` it's listening to changing.

Reach for `StatelessWidget` by default. Only step up to [[StatefulWidget]] when a widget needs to remember something *between* rebuilds that isn't just handed to it from outside — a scroll position, a text controller, an animation, a flag like "is this menu open."

## In the nested app

`lib/widgets/empty_state.dart` is about as pure an example as it gets — two fields in, a `Column` out, nothing else:

```dart
class EmptyState extends StatelessWidget {
  final IconData icon;
  final String message;

  const EmptyState({super.key, required this.icon, required this.message});

  @override
  Widget build(BuildContext context) {
    final colors = Theme.of(context).colorScheme;
    final text = Theme.of(context).textTheme;
    return Center(
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Icon(icon, size: 40, color: colors.primary.withValues(alpha: 0.25)),
          const SizedBox(height: 12),
          Text(message, style: text.bodyMedium?.copyWith(
            color: colors.onSurface.withValues(alpha: 0.35),
          )),
        ],
      ),
    );
  }
}
```

`folder_screen.dart` decides *which* icon and message to pass based on whether the current folder is root or a subfolder — `EmptyState` itself doesn't know or care why it's empty, it just renders what it's told:

```dart
return EmptyState(
  icon: isRoot ? Icons.inbox_rounded : Icons.create_new_folder_outlined,
  message: isRoot ? 'Your workspace is empty' : 'This folder is empty',
);
```

That's the pattern to imitate: push decisions up to whoever has the context to make them, and keep leaf widgets dumb and reusable. Note `EmptyState` still reads `Theme.of(context)` — being stateless doesn't mean "ignores `BuildContext`," it means "has no private mutable fields." See [[BuildContext]].

## Key takeaways

- Always give a `StatelessWidget` a `const` constructor when possible (`const EmptyState(...)`) — Flutter can skip rebuilding a `const` widget entirely if its parent rebuilds but its arguments haven't changed.
- If you find yourself wanting a field that changes after construction without a parent rebuild triggering it, that's the signal you actually need [[StatefulWidget]].
- Being stateless doesn't limit what you can *read* (theme, inherited data) — only what you can *hold* privately across rebuilds.
