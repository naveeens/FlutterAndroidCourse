# Adaptive layouts

"Adaptive" (as distinct from [[Responsive layouts]]) usually means: pick a fundamentally different layout or widget set for a fundamentally different device class or platform, rather than fluidly resizing the same layout. A phone gets a bottom nav bar and single-column list; a tablet gets a side rail and two-pane master-detail. A phone dialog might become a full-screen page on a much smaller device. This is typically driven by `LayoutBuilder` (react to the actual constraints your widget was given) or breakpoints on `MediaQuery.size.width`.

## Why the nested app doesn't need this — and one place it almost does

The nested app is phone-first with a single-column tree view; it doesn't ship tablet-specific layouts, so there's no `LayoutBuilder`-driven breakpoint logic to point to as a real example.

The closest thing to adaptive behavior in the codebase is platform-adaptive *widgets* rather than platform-adaptive *layout*: `CircularProgressIndicator.adaptive()` in `lib/folder_screen.dart` and `lib/bin_screen.dart`:

```dart
if (notifier.loading) {
  return const Center(child: CircularProgressIndicator.adaptive());
}
```

`.adaptive()` constructors (also available on `Switch`, `Slider`, and a few others) render a Material spinner on Android and a Cupertino-style spinner on iOS automatically — a small, targeted form of "adapt to the platform" that doesn't require any layout-level branching, just picking the right constructor.

## What real adaptive layout code looks like, for reference

Since the app doesn't need it, here's the shape you'd reach for if it did — worth knowing conceptually even without an in-app example:

```dart
LayoutBuilder(
  builder: (context, constraints) {
    if (constraints.maxWidth >= 840) {
      return TwoPaneLayout(list: nodeList, detail: noteEditor);
    }
    return nodeList; // phone: single column, navigate to a new screen for detail
  },
)
```

The nested app's actual navigation model — pushing `NoteEditorScreen` as a full route via `Navigator.push` from `folder_screen.dart`'s `_handleTap` — is exactly the "phone" branch of that pattern. If this app ever grew a tablet layout, the natural adaptive move would be swapping that `Navigator.push` for an inline detail pane above some width threshold, without changing `NoteEditorScreen`'s own content at all.

## Key takeaways

- Adaptive = different layout/widgets per device class; responsive = same layout, fluidly resized. The nested app only needs the latter, plus a couple of `.adaptive()` widget constructors for platform-appropriate visuals.
- `LayoutBuilder` reads the *actual constraints* a widget was given (which can differ from full screen size if it's nested inside something else), making it more reliable than raw `MediaQuery.size` for layout decisions scoped below the app root.
- `.adaptive()` constructors are the cheapest form of platform adaptation — reach for them before hand-rolling `Platform.isIOS` branches for widgets that already support it.
