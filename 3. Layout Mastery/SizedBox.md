# SizedBox

`SizedBox` forces its child (or, with no child, itself) to a specific width and/or height. It has two everyday jobs: fixed spacing between elements in a `Row`/`Column`, and forcing a size onto a widget that would otherwise size itself differently.

## In the nested app

**As a spacer** — by far the most common use, all over the app:

```dart
Icon(Icons.folder_rounded, size: 20, color: accent),
if (isFolder && node.isPinned) ...[
  const SizedBox(width: 4),
  Icon(Icons.push_pin_rounded, size: 12, color: accent),
],
const SizedBox(width: 10),
Expanded(child: Text(node.name, ...)),
```

(`lib/widgets/node_tile.dart`) — three different gap sizes (4, 10, and elsewhere 8, 12, 16, 24) used deliberately to create visual rhythm between icon, pin badge, and title.

**As a size constraint on something that wouldn't otherwise be that size** — the voice recording indicator in `note_editor_screen.dart`:

```dart
Icon(icon, color: iconColor, size: 22)
```

and the loading spinner inside a button:

```dart
child: importing
    ? const SizedBox(
        width: 16,
        height: 16,
        child: CircularProgressIndicator(strokeWidth: 2),
      )
    : const Text('Import'),
```

(`folder_screen.dart`, GitHub import sheet). A bare `CircularProgressIndicator` defaults to a fairly large size — wrapping it in a 16×16 `SizedBox` shrinks it to fit inside a `FilledButton` alongside regular text-sized content, without needing a whole separate small-spinner widget.

`SizedBox.shrink()` also appears as an explicit "render nothing, but participate in layout as a zero-size widget" — used in `node_tile.dart`'s folder count `Consumer` when there's no count to show yet:

```dart
if (counts == null) return const SizedBox.shrink();
```

## Key takeaways

- `const SizedBox(width: N)` / `const SizedBox(height: N)` for spacing is idiomatic and cheap — always mark it `const` since it never changes.
- `SizedBox` around a child forces that child's size regardless of what the child would naturally choose — useful for shrinking oversized default widgets like `CircularProgressIndicator`.
- `SizedBox.shrink()` is the standard "return nothing but keep the widget non-null" pattern, preferable to returning `Container()` (which is heavier and less explicit about intent).
