# Material You

Material You (Android 12+) is Google's dynamic-color system — the OS extracts a color palette from the user's wallpaper and makes it available to apps via `dynamicLightColorScheme()`/`dynamicDarkColorScheme()` (in `com.google.android.material` / Compose Material 3), so an app's UI can automatically match the rest of the user's home screen without the app author picking any specific colors themselves.

## Not used in the nested app — it has its own, deliberately explicit theming system instead

This app doesn't hook into Android's dynamic color APIs at all. Instead, it has a hand-built, user-controlled theming system: `lib/app_theme.dart`'s `AppThemes` defines a fixed set of theme keys (`'light'`, `'dark'`, `'amoled'`) plus a separately user-chosen accent color, persisted via [[SharedPreferences]] (`lib/main.dart`'s `_prefKey`/`_accentPrefKey`) and pushed to the native widget via [[MethodChannel]] so the two stay in sync:

```dart
final savedAccent = prefs.getInt(_accentPrefKey);
final initialAccent = savedAccent != null ? Color(savedAccent) : AppThemes.defaultAccent;
unawaited(WidgetBridge.updateTheme(initialTheme, initialAccent.value));
```

The native side, `WidgetTheme.kt`, mirrors this exact scheme rather than reading anything from the OS's dynamic color system:

```kotlin
object WidgetTheme {
    fun resolve(context: Context): WidgetThemeColors = when (WidgetPrefs.getThemeKey(context)) {
        "dark" -> WidgetThemeColors(backgroundDrawableRes = R.drawable.widget_bg_dark, textColor = 0xFFE6E1E5.toInt(), ...)
        "amoled" -> WidgetThemeColors(backgroundDrawableRes = R.drawable.widget_bg_amoled, ...)
        else -> WidgetThemeColors(backgroundDrawableRes = R.drawable.widget_bg_light, ...) // "light"
    }
    fun accentColor(context: Context): Int = WidgetPrefs.getAccentColor(context)
}
```

That's a deliberate design choice, not a gap: this app wants the user to pick *their* accent color explicitly (via the in-app color wheel — see `_showAccentPicker` in `lib/folder_screen.dart`) and have it apply consistently everywhere — in-app and in the widget — regardless of what wallpaper is set. Material You's dynamic-color extraction would actually work *against* that goal, since it ties the app's colors to the wallpaper rather than to a color the user picked specifically for this app.

## Where Material You would be a reasonable addition

If this app wanted an *additional* theme option — "match my wallpaper" as one more choice alongside light/dark/amoled/custom-accent, rather than replacing the existing system — it could read the dynamic color scheme where available and offer it as an opt-in theme key, falling back to the current fixed palette on pre-Android-12 devices or when the user prefers their own accent choice.

## Key takeaways

- Material You's dynamic color is a real, useful default for apps that want to visually blend into the OS with minimal design effort — but it's opt-in for a reason: apps with a strong, deliberate visual identity (or, as here, ones that let the user pick their own accent) often choose not to use it.
- This app's `AppThemes`/`WidgetTheme` pairing shows the alternative done well: explicit, user-controlled theming kept in sync between the Flutter UI and native widget via a small persisted theme key plus accent color, rather than either side improvising its own colors.
- Choosing not to adopt a platform convention (dynamic color) is a legitimate design decision when it conflicts with a more specific goal (consistent, user-chosen branding) — worth recognizing as a choice, not an oversight, when reading this app's theming code.
