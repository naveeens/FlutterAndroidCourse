# Platform Channels

A platform channel is Flutter's general mechanism for crossing the boundary between Dart code and native platform code (Kotlin/Java on Android, Swift/Objective-C on iOS) — see [[Flutter architecture]] for why that boundary exists at all (the Flutter engine draws its own UI and doesn't wrap native widgets, so anything genuinely platform-specific has to be explicitly bridged). Every channel is identified by a unique string name and a codec (usually the standard binary codec, which handles common types — strings, numbers, lists, maps — transparently). Three concrete channel types build on this: [[MethodChannel]] (request/response, "call this native method"), `EventChannel` (a continuous stream of native-to-Dart events — see [[EventChannel]] for why this app rolls its own alternative instead), and `BasicMessageChannel` (raw message passing, rarer in application code).

## In the nested app

This app uses two `MethodChannel`s, both declared with the same naming convention — a reverse-domain-style string that must match *exactly* on both the Dart and Kotlin sides:

```dart
// lib/services/widget_bridge.dart
static const _channel = MethodChannel('live.suture.nested/widget');

// lib/services/widget_nav_service.dart
static const MethodChannel _channel = MethodChannel('live.suture.nested/widget_nav');
```

```kotlin
// MainActivity.kt
MethodChannel(flutterEngine.dartExecutor.binaryMessenger, "live.suture.nested/widget")
    .setMethodCallHandler { call, result -> ... }

val navChannel = MethodChannel(flutterEngine.dartExecutor.binaryMessenger, "live.suture.nested/widget_nav")
navChannel.setMethodCallHandler { call, result -> ... }
```

Two separate channels for two separate concerns — one purely Dart-to-native (telling the widget to refresh/re-theme), one bidirectional (Dart pulling a cold-start folder id, native pushing a hot-start folder id) — is a deliberate, sensible split rather than one do-everything channel. `MainActivity.configureFlutterEngine` is where both are registered, which matters: this is the one guaranteed point where a fresh `FlutterEngine` exists and channel handlers must be (re-)attached, called every time the engine (re)initializes.

## Key takeaways

- The channel name string is the entire contract between Dart and native — it must match character-for-character on both sides, and there's no compile-time check that catches a mismatch (a typo just means the native handler never fires, silently).
- `flutterEngine.dartExecutor.binaryMessenger` is the actual transport a channel is built on, registered once per engine instance in `configureFlutterEngine` — see [[Flutter architecture]] for where that fits in the engine/embedder layering.
- Pick `MethodChannel` for request/response or fire-and-forget calls (this app's whole usage), `EventChannel` for a genuine continuous native-originated stream — see [[EventChannel]] for why this app's one stream-like need (widget-tap events) is instead built from a plain `MethodChannel` plus a Dart-side `StreamController`.
- See [[MethodChannel]] for the call/response mechanics in detail, and [[MainActivity.kt's role]] — covered under [[Activity lifecycle]] — for where channel registration lives in this app.
