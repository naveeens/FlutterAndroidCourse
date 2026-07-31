# EventChannel

`EventChannel` is the streaming counterpart to [[MethodChannel]] — instead of one call/one response, native code can push a continuous sequence of events to Dart over time, surfaced on the Dart side as a `Stream`. It's the natural fit for genuinely ongoing native data: sensor readings, location updates, or (as this app's actual use case would suggest) a sequence of native "the user did X" events.

## Not used in the nested app — it rolls the equivalent by hand instead

The widget-tap-while-app-running scenario is exactly what `EventChannel` is designed for — an unbounded sequence of native events ("folder N was tapped") that Dart should observe as a stream. But `lib/services/widget_nav_service.dart` doesn't use `EventChannel` at all; it builds the same shape manually, combining a plain [[MethodChannel]] with a Dart-side `StreamController`:

```dart
class WidgetNavService {
  static const MethodChannel _channel = MethodChannel('live.suture.nested/widget_nav');
  final StreamController<int> _controller = StreamController<int>.broadcast();

  Stream<int> get openFolderStream => _controller.stream;

  void init() {
    _channel.setMethodCallHandler((call) async {
      if (call.method == 'onOpenFolder') {
        final id = call.arguments as int?;
        if (id != null) _controller.add(id);
      }
    });
  }
}
```

Native code fires this by calling `invokeMethod` on the same channel (`MainActivity.kt`'s `onNewIntent`), rather than an `EventChannel`'s dedicated sink API:

```kotlin
widgetNavChannel?.invokeMethod("onOpenFolder", it)
```

This is a reasonable, deliberate simplification rather than an oversight: `EventChannel` is built for genuinely high-frequency or long-lived event streams (sensor data arriving many times a second) where its more structured setup — a native `StreamHandler` with `onListen`/`onCancel` lifecycle callbacks tied to whether Dart is actually listening — earns its complexity. This app's event is rare (a widget tap happens once per user action, not continuously) and this app only ever needs *one* Dart-side listener (`FolderScreen`'s `initState`, see [[Streams]]) — a hand-rolled `MethodChannel` + `StreamController` gets the same observable-stream ergonomics on the Dart side with meaningfully less native-side ceremony.

## What the EventChannel version would look like, for comparison

```dart
static const _eventChannel = EventChannel('live.suture.nested/widget_nav_events');
Stream<int> get openFolderStream => _eventChannel.receiveBroadcastStream().cast<int>();
```

```kotlin
EventChannel(flutterEngine.dartExecutor.binaryMessenger, "live.suture.nested/widget_nav_events")
    .setStreamHandler(object : EventChannel.StreamHandler {
        override fun onListen(arguments: Any?, events: EventChannel.EventSink) {
            eventSink = events // stash for later .success(folderId) calls
        }
        override fun onCancel(arguments: Any?) {
            eventSink = null
        }
    })
```

The meaningful difference: `EventChannel.StreamHandler`'s `onListen`/`onCancel` give native code an explicit signal for "Dart actually has a listener right now" versus "no one's listening, don't bother computing/sending events" — a real advantage for expensive or high-frequency native event sources, but not something this app's rare, cheap widget-tap events need.

## Key takeaways

- `EventChannel` earns its complexity for high-frequency, long-lived native event streams with real backpressure/lifecycle concerns — a rare, occasional event (like this app's widget taps) doesn't need it.
- A `MethodChannel` (native → Dart `invokeMethod` calls) plus a Dart-side `StreamController.broadcast()` is a legitimate, lighter-weight substitute when you just want stream-shaped ergonomics on the Dart side without native-side stream lifecycle management.
- If this app's widget interaction volume or event frequency ever grew significantly, `EventChannel`'s `onListen`/`onCancel` would become worth adopting specifically to avoid native code doing work for events nobody's listening to.
