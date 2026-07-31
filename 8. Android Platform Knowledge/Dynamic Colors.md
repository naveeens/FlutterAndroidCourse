# Dynamic Colors

"Dynamic colors" is the general capability underlying [[Material You]] — colors computed/derived at runtime (from wallpaper, from a seed color, from system state) rather than fixed at design time. It's worth treating as a distinct, broader concept from Material You specifically, because an app can build its *own* dynamic color system without ever touching Android's wallpaper-extraction APIs — which is exactly what this app does.

## In the nested app: dynamic colors without Material You

This app has real dynamic color logic — colors computed at runtime from user choices and per-item state — just sourced from its own accent-color system rather than from the OS. `lib/app_theme.dart`'s `AppColors.themedFromPriorityOrNull` is the core of it: a task/folder's display color is derived from its `priority` value and the *current* theme's color scheme, not a hardcoded color:

```dart
final nodeColor = AppColors.themedFromPriorityOrNull(node.priority, Theme.of(context).colorScheme);
final accent = nodeColor ?? colors.primary;
```

Because this reads `Theme.of(context).colorScheme` — itself built from the user's chosen accent color (see [[Material You]] for how that accent gets set and persisted) — every priority-colored dot, icon, and border in the app recomputes correctly the instant the user picks a new accent, with zero hardcoded color literals scattered through widget code.

The native widget side mirrors this same computed-from-state approach rather than hardcoding colors, in `WidgetTheme.kt`:

```kotlin
fun priorityColor(context: Context, priority: Int): Int = when (priority) {
    1 -> context.getColor(R.color.priority_blue)
    2 -> context.getColor(R.color.priority_yellow)
    3 -> context.getColor(R.color.priority_red)
    else -> accentColor(context) // priority 0 ("base") = the user's chosen accent
}
```

And `NestedWidgetProvider.kt`'s header colors are resolved dynamically per-refresh from the currently stored theme key, not fixed at compile time:

```kotlin
val theme = WidgetTheme.resolve(context)
rv.setInt(R.id.widget_root, "setBackgroundResource", theme.backgroundDrawableRes)
rv.setTextColor(R.id.widget_header_title, theme.textColor)
```

## The distinction worth keeping straight

"Dynamic" here means "computed from current state at the point of use," not "derived from the OS's wallpaper-extraction feature" — those are two separable ideas that happen to share a name in Android's own Material You branding. This app is a clean example of the former without the latter: real runtime color computation (priority → color, theme key → palette), deliberately decoupled from Android's specific wallpaper-based mechanism, discussed in [[Material You]].

## Key takeaways

- Dynamic color computation (deriving a color from current state — user preference, item metadata, theme mode) is a broader, more general idea than Material You's specific wallpaper-extraction feature — this app fully embraces the former while opting out of the latter.
- Centralizing the derivation logic (`AppColors.themedFromPriorityOrNull` on the Dart side, `WidgetTheme.priorityColor` on the native side) rather than scattering `if (priority == 1) Colors.blue` checks through every widget keeps the app's and widget's color logic consistent and easy to change in one place.
- See [[Material You]] for why this app deliberately doesn't use Android's OS-level dynamic color extraction despite having its own dynamic color system.
