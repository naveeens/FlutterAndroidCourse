# SharedPreferences

`SharedPreferences` is a simple persistent key-value store (backed by `UserDefaults` on iOS, `SharedPreferences` on Android) for small pieces of data — settings, flags, the last-used value of something. It's not for structured or relational data (that's what [[SQLite]] is for in this app) — just individual named values that are cheap to read and write.

## In the nested app

`lib/main.dart` uses it for exactly what it's meant for: the user's chosen theme and accent color, read synchronously before the first frame so there's no visual flash:

```dart
const _prefKey = 'theme_key';
const _accentPrefKey = 'accent_color';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  ...
  final prefs = await SharedPreferences.getInstance();
  final savedTheme = prefs.getString(_prefKey);
  final initialTheme = (savedTheme != null && AppThemes.themeKeys.contains(savedTheme)) ? savedTheme : 'light';

  final savedAccent = prefs.getInt(_accentPrefKey);
  final initialAccent = savedAccent != null ? Color(savedAccent) : AppThemes.defaultAccent;
  ...
}
```

Two things worth noting: `getString`/`getInt` return `null` when the key has never been set (first launch), which is why every read here is paired with a fallback (`'light'`, `AppThemes.defaultAccent`); and `savedTheme` is validated against `AppThemes.themeKeys.contains(savedTheme)` before being trusted — defensive against a value that was valid in an older app version but no longer corresponds to a real theme.

Writing back happens in `_NestedAppState`, whenever the user changes a setting:

```dart
Future<void> _saveTheme(String key) async {
  final prefs = await SharedPreferences.getInstance();
  await prefs.setString(_prefKey, key);
  unawaited(WidgetBridge.updateTheme(key, _accent.value));
}

Future<void> _saveAccent(Color color) async {
  final prefs = await SharedPreferences.getInstance();
  await prefs.setInt(_accentPrefKey, color.value);
  unawaited(WidgetBridge.updateTheme(_themeKey, color.value));
}
```

Note the UI doesn't wait for the save to land before reflecting the change — `setState` in the caller (`FolderScreen`'s `onThemeChanged`/`onAccentChanged` callbacks) updates in-memory state immediately, and `_saveTheme`/`_saveAccent` persist it in the background. This is the right call for a setting like theme: the user should see the change instantly, and losing an in-flight preference write on an app crash mid-save is an acceptable, extremely rare cost for the responsiveness gained.

## Key takeaways

- `SharedPreferences.getInstance()` is async and (internally) cached — calling it repeatedly is cheap, but the *first* call in `main()` should be awaited before the UI that depends on it renders, exactly as this app does for the initial theme.
- Every getter (`getString`, `getInt`, `getBool`, ...) returns `null` on a missing key — always have a real fallback value, never assume a key exists just because your code expects it to.
- Right-sized for flags and simple settings; once you need relationships, queries, or anything beyond "one key → one value," reach for [[SQLite]] instead, as this app does for its actual folder/task data.
