# Hot reload vs restart

**Hot reload** injects your updated Dart source into the already-running Dart VM and rebuilds the widget tree, but keeps the app's existing state (whatever's currently in memory — your `State` objects' fields, navigation stack, scroll positions). It's fast because it skips restarting the whole app and re-running `main()`.

**Hot restart** tears down the Dart VM state and starts over — `main()` runs again from scratch, all state is lost, but it's still faster than a full rebuild/reinstall because the compiled app package itself doesn't need to be rebuilt.

A **full restart** (uninstall/reinstall or `flutter run` from cold) is the only way to pick up changes to anything outside the Dart VM's reach: native code, `pubspec.yaml` dependencies, `AndroidManifest.xml`, or asset declarations.

## Why the nested app makes this concrete

Try this in the running app: change a `Text` string or a color in `lib/widgets/node_tile.dart`, save, hot reload. The currently-open folder, scroll position, and any in-progress edit stay exactly where they were — only the visual change appears. That's hot reload's whole value proposition: fast iteration on UI without losing your place.

Now change something in `lib/main.dart`'s `void main()` — say, the initial theme fallback logic:

```dart
final initialTheme = (savedTheme != null && AppThemes.themeKeys.contains(savedTheme)) ? savedTheme : 'light';
```

Hot reload will *not* re-run this — `main()` only executes once, at cold start. You need a **hot restart** to see a change here take effect, because it re-runs `main()` from the top, including the `SharedPreferences` read and the `runApp` call.

Now go further: edit `android/app/src/main/kotlin/live/suture/nested/widget/NestedWidgetProvider.kt` (the native widget code) or add a dependency to `pubspec.yaml`. Neither hot reload nor hot restart will pick this up — you need a **full stop and `flutter run` again**, because these live outside the running Dart VM entirely: native code needs recompiling by Gradle, and new packages need fetching and linking before the VM even starts.

## Key takeaways

- Hot reload: fastest, preserves state, only affects Dart code already loaded and running — great for tweaking `build()` methods, styling, layout.
- Hot restart: loses state, still Dart-only, needed when changed code wouldn't naturally re-run again (like `main()`, top-level initialization, or anything involving `const` values changing).
- Full restart: required for native code changes (`android/`, `ios/`), new/changed dependencies in `pubspec.yaml`, and manifest/plist changes.
- If a hot reload "does nothing" or behaves oddly, hot restart first before assuming there's a bug — stale in-memory state is a common false alarm.
