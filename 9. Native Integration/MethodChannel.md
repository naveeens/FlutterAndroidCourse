# MethodChannel

`MethodChannel` is the request/response flavor of [[Platform Channels]] — Dart calls `channel.invokeMethod('name', args)` and gets back a `Future` that resolves (or throws) once native code responds; native code registers a handler that receives a method name plus arguments and calls back `result.success(...)` / `result.error(...)`. It's genuinely symmetric — either side can also *initiate* a call to the other (Dart→native and native→Dart both use the same channel object, just calling `invokeMethod` from whichever side is the caller for that particular message).

## Dart calling native: WidgetBridge

`lib/services/widget_bridge.dart` is the simplest possible shape — call a native method, ignore (or swallow) the result:

```dart
class WidgetBridge {
  static const _channel = MethodChannel('live.suture.nested/widget');

  static Future<void> refreshWidget() async {
    try {
      await _channel.invokeMethod('refreshWidget');
    } on PlatformException catch (_) {
      // Non-fatal — widget just won't be up to date until next trigger.
    } on MissingPluginException catch (_) {
      // No native handler on non-Android platforms — nothing to do.
    }
  }

  static Future<void> updateTheme(String themeKey, int accentColorValue) async {
    try {
      await _channel.invokeMethod('updateTheme', {'themeKey': themeKey, 'accent': accentColorValue});
    } on PlatformException catch (_) {
    } on MissingPluginException catch (_) {
    }
  }
}
```

Both exception types matter for different reasons: `PlatformException` is a real error the native handler explicitly threw (or an argument mismatch); `MissingPluginException` means there's *no handler at all* for this channel — which happens legitimately on platforms with no matching native implementation (this app is Android-only in practice, but the defensive catch means calling these methods never crashes even hypothetically). The `unawaited(WidgetBridge.refreshWidget())` calls sprinkled through `NestedTreeNotifier` (see [[ChangeNotifier]]) are deliberate — the app doesn't want a slow or failed widget refresh to block or fail a local database write.

The native side, in `MainActivity.kt`, dispatches by method name:

```kotlin
MethodChannel(flutterEngine.dartExecutor.binaryMessenger, "live.suture.nested/widget")
    .setMethodCallHandler { call, result ->
        when (call.method) {
            "refreshWidget" -> {
                NestedWidgetProvider.refreshAllWidgets(applicationContext)
                result.success(null)
            }
            "updateTheme" -> {
                val themeKey = call.argument<String>("themeKey") ?: "light"
                val accent = (call.argument<Number>("accent"))?.toInt() ?: WidgetPrefs.DEFAULT_ACCENT
                WidgetPrefs.setTheme(applicationContext, themeKey, accent)
                NestedWidgetProvider.refreshAllWidgets(applicationContext)
                result.success(null)
            }
            else -> result.notImplemented()
        }
    }
```

The `accent` argument reading is worth studying closely — `call.argument<Number>("accent"))?.toInt()`, not `call.argument<Int>("accent")` directly, with a comment explaining why: *"Opaque ARGB color ints (alpha=0xFF) exceed Int32.MAX, so the platform channel codec may deliver this as a Long, not an Int."* This is a real, easy-to-miss gotcha of crossing the platform channel boundary — Dart's `int` doesn't distinguish 32-bit/64-bit the way Kotlin's `Int`/`Long` do, so a value that looks like a plain `int` in Dart can arrive as either type on the native side depending on its magnitude; reading it as the common supertype `Number` and converting explicitly sidesteps a runtime `ClassCastException`.

## Native calling Dart: WidgetNavService

`widget_nav_service.dart` shows the *other* direction — Dart registers a handler for calls initiated from native:

```dart
_channel.setMethodCallHandler((call) async {
  if (call.method == 'onOpenFolder') {
    final id = call.arguments as int?;
    if (id != null) _controller.add(id);
  }
});
```

and `MainActivity.kt`'s `onNewIntent` initiates it:

```kotlin
widgetNavChannel?.invokeMethod("onOpenFolder", it)
```

Plus a genuine request/response call in the opposite direction, Dart asking native for a stashed value:

```dart
Future<int?> consumeInitialFolderId() {
  return _channel.invokeMethod<int>('getInitialFolderId');
}
```

```kotlin
"getInitialFolderId" -> {
    result.success(pendingFolderId)
    pendingFolderId = null
}
```

## Key takeaways

- Same `MethodChannel` object, either side can call `invokeMethod` — direction isn't fixed by the channel itself, only by which side happens to initiate a given message.
- Wrap every `invokeMethod` call in a `try`/`catch` for `PlatformException` (a real native-side error) and `MissingPluginException` (no handler registered at all) — this app never lets a channel failure propagate up and break a local operation.
- Numeric types crossing the channel can silently change representation (`int` vs. `Long`) depending on magnitude — reading as the common numeric supertype and converting explicitly, as this app's `updateTheme` handler does, avoids a crash.
- `call.argument<T>("key")` returns `null` for a missing/wrong-type key rather than throwing — always pair it with a sensible fallback (`?: "light"`, as seen above), never assume the argument is present.
