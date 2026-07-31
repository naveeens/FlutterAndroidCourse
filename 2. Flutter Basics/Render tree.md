# Render tree

The render tree (made of `RenderObject`s) is where actual layout and painting happen. Each `RenderObject` knows how to size itself given constraints from its parent, position its children, and paint pixels. It's the layer below the [[Element tree]] — most widgets you write (`Padding`, `Row`, `Container`) don't paint anything themselves; they configure a `RenderObject` that does.

Layout in Flutter flows in one direction: constraints go *down* the tree (a parent tells its child "you may be at most this big"), sizes go *back up* (a child tells its parent "I've decided to be this big, within what you allowed"), and the parent then positions its children. This is normally summarized as "constraints go down, sizes go up, parent sets position." It's why a widget can't just declare its own size unconditionally — it has to negotiate within whatever box its parent handed it. See [[Constraints]].

## In the nested app

`lib/widgets/node_tile.dart` layers several render-affecting widgets to build one row:

```dart
Container(
  decoration: BoxDecoration(
    color: widget.selected ? colors.primaryContainer.withValues(alpha: 0.25) : colors.surface,
    border: Border(
      bottom: BorderSide(color: colors.outline.withValues(alpha: 0.08), width: 1),
      left: BorderSide(color: widget.selected ? colors.primary : Colors.transparent, width: 4),
    ),
  ),
  child: widget.isEditing ? AddItemBar(...) : _buildNormalView(context),
)
```

Each of these — `Container`'s implicit `DecoratedBox` + `Padding`, the `Row`/`Column` inside `_buildNormalView`, `Expanded` around the title `Text` — corresponds to a `RenderObject` that participates in layout. `Expanded` in particular only makes sense in terms of the render tree: it tells the parent `Row`'s render object "give me a share of the remaining horizontal space after fixed-size siblings are laid out," which is a render-tree-level negotiation, not something the widget itself computes.

Repainting without relayout is the cheap case Flutter optimizes for — e.g., `NodeTile`'s selected/unselected background swap only changes a `Container`'s paint color, not its size, so Flutter can skip a full layout pass and just repaint that `RenderObject`.

## Key takeaways

- Widgets are descriptions; `RenderObject`s are what actually measure, position, and paint. Most layout widgets (`Row`, `Padding`, `Align`) exist purely to configure one.
- Layout is a single top-down-then-bottom-up pass per frame: constraints flow down, sizes flow up, then the parent positions children. A child can never demand a size larger than what its parent's constraints allow.
- Distinguishing "needs relayout" from "just needs repaint" is a real performance lever — see [[Widget rebuild optimization]] and [[Frame rendering]].
