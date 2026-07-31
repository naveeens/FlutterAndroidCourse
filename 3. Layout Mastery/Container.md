# Container

`Container` is a convenience widget that bundles padding, margin, decoration (background color, border, border radius, shadow), constraints, and alignment into one widget — under the hood it composes several single-purpose widgets (`Padding`, `DecoratedBox`, `ConstrainedBox`, `Align`) for you. That convenience is also the reason to be deliberate about it: if all you need is a background color, `ColoredBox` is cheaper; if all you need is padding, `Padding` is more explicit about intent. Reach for `Container` when you actually need several of those things at once, which in practice is often.

## In the nested app

`lib/widgets/node_tile.dart` uses `Container` for exactly what it's good at — decoration that depends on state:

```dart
Container(
  decoration: BoxDecoration(
    color: widget.selected
        ? colors.primaryContainer.withValues(alpha: 0.25)
        : colors.surface,
    border: Border(
      bottom: BorderSide(color: colors.outline.withValues(alpha: 0.08), width: 1),
      left: BorderSide(
        color: widget.selected ? colors.primary : Colors.transparent,
        width: 4,
      ),
    ),
  ),
  child: widget.isEditing ? AddItemBar(...) : _buildNormalView(context),
)
```

Both the fill color and the left border color are computed from `widget.selected` — that 4px colored left border is the row's whole "this is selected" affordance, achieved with zero extra widgets, just a conditional inside `BoxDecoration`.

`note_editor_screen.dart`'s formatting toolbar shows `Container` used for size + decoration together:

```dart
Container(
  height: 48,
  width: double.infinity,
  decoration: BoxDecoration(
    color: colors.surfaceContainerHighest.withValues(alpha: 0.85),
    border: Border(top: BorderSide(color: colors.onSurface.withValues(alpha: 0.1))),
  ),
  child: Row(...),
)
```

Here `Container` is doing three jobs in one widget: fixing the toolbar's height, giving it a translucent background, and drawing a hairline top border — each of which would otherwise need its own wrapper widget.

## Key takeaways

- `Container` composes multiple concerns (size, padding, margin, decoration, alignment) — great when you need several together, unnecessary overhead when you only need one (prefer `Padding`, `SizedBox`, or `ColoredBox` for a single concern).
- `BoxDecoration` inside a `Container` is where most conditional, state-driven styling lives in this app — see the selected-row border above.
- A `Container` with only a `color` and a `child` still allocates a `DecoratedBox` under the hood; `ColoredBox` is the leaner choice when decoration is *just* a solid color with no border/radius/shadow.
