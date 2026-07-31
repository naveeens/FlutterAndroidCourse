# ADB

ADB (Android Debug Bridge) is the command-line tool for talking to a connected device or emulator directly — installing/uninstalling packages, viewing logs, inspecting app state, sending simulated input, and more. `flutter run` uses it under the hood, but knowing raw `adb` commands matters most exactly when something *outside* Flutter's own tooling needs inspecting — which, for this app, means its native Android widget.

## Where ADB is genuinely useful for this app specifically

Most day-to-day Flutter development (hot reload, `flutter logs`, the DevTools inspector — see [[Debugging widgets]]) doesn't need raw `adb`. But this app ships real native Android components — a home-screen `AppWidgetProvider`, broadcast receivers, a `RemoteViewsService` (see [[Widgets (App Widgets)]]) — none of which show up in Flutter's own debugging tools, since none of them run inside the Flutter engine at all.

**Watching native logs** while interacting with the widget (something `flutter logs` won't surface, since it filters to Flutter/Dart output):

```
adb logcat | grep -i "live.suture.nested"
```

Every `e.printStackTrace()` call in `NodeDbHelper.kt` (there are several, guarding SQLite operations run from receiver/service code) lands here — that's the only place to see them, since a `BroadcastReceiver` or `RemoteViewsFactory` has no Flutter/Dart console to print to.

**Inspecting the app's actual SQLite file on-device** — useful for confirming the native widget code and the Flutter app are really reading/writing the same database (`NodeDbHelper.kt`'s docstring specifically notes it opens "the SAME sqlite file the Flutter app's `sqflite` plugin writes to"):

```
adb shell run-as live.suture.nested ls databases/
adb exec-out run-as live.suture.nested cat databases/nested.db > nested.db
```

**Simulating a widget-triggered app launch** without physically placing and tapping a widget, by sending the same broadcast the system would send:

```
adb shell am broadcast -a android.appwidget.action.APPWIDGET_UPDATE -n live.suture.nested/.widget.NestedWidgetProvider
```

**Clearing app data** to test a truly fresh install (first-run `SharedPreferences` behavior in `lib/main.dart`, the `_onCreate` SQLite path in `DatabaseHelper` rather than `_onUpgrade` — see [[Data migration]]) without uninstalling/reinstalling:

```
adb shell pm clear live.suture.nested
```

## Key takeaways

- `flutter run`/`flutter logs`/DevTools cover Dart-side debugging well; raw `adb` becomes necessary the moment you're debugging something native — this app's widget provider, receivers, and `RemoteViewsService` all qualify.
- `adb logcat` is the only way to see native-side `Log`/`printStackTrace` output — Flutter's own log stream doesn't include it.
- `adb shell run-as <applicationId>` gives shell access scoped to a debuggable app's private data (its databases, shared prefs) without root — useful for confirming exactly what's actually stored on-device, as opposed to what the code claims.
- `adb shell pm clear <applicationId>` is a faster, more reliable way to simulate "fresh install" than uninstall/reinstall for testing first-run and migration code paths.
