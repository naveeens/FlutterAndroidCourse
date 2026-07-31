# Streams

A `Stream` is an asynchronous sequence of events over time — think of a `Future` that can complete more than once. You get values out of one by `listen()`ing (with a callback per event), by `await for` in an async function, or via `StreamBuilder` in the widget tree. A `StreamController` is how you create your own stream to push events into.

## In the nested app

`lib/services/widget_nav_service.dart` is the app's one real hand-rolled stream — it bridges "the user tapped a folder on the home-screen widget" (a native Android event, arriving via [[MethodChannel]]) into something Dart code elsewhere can react to:

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
  ...
}
```

`.broadcast()` matters here — a plain (single-subscription) `StreamController` only allows one listener ever; `broadcast()` allows multiple, or none, or a listener that comes and goes as `FolderScreen` is created and disposed across the app's lifetime. Every widget-tap event received from the platform channel is pushed in via `_controller.add(id)`.

The consumer side, in `lib/folder_screen.dart`, is a plain `listen()` subscription stored so it can be cancelled later:

```dart
StreamSubscription<int>? _widgetNavSub;

@override
void initState() {
  super.initState();
  _widgetNavSub = WidgetNavService.instance.openFolderStream.listen(_handleOpenFolder);
  ...
}

@override
void dispose() {
  _widgetNavSub?.cancel();
  _scrollController.dispose();
  super.dispose();
}
```

This is the standard shape for consuming a stream from a `StatefulWidget`: subscribe in `initState`, store the `StreamSubscription`, cancel it in `dispose`. Skipping the cancel would leak — the subscription (and its closure over `this`) would keep the disposed `State` alive in memory and could call `_handleOpenFolder` (which checks `mounted` defensively) after the widget's gone.

## Key takeaways

- `StreamController.broadcast()` when more than one listener might attach (or the app's widget tree might rebuild the listener); a single-subscription controller throws on a second `listen()`.
- Every `stream.listen()` returns a `StreamSubscription` — always store it and `.cancel()` it in `dispose()`, exactly like `_widgetNavSub` above.
- Streams are the natural fit for "events arriving from outside Dart's normal call/return flow" — platform channel callbacks (this example), user input events, or any producer that emits 0-to-many values over time. For state that has a single current value plus change notifications, [[ChangeNotifier]]/[[ValueNotifier]] are usually a better fit than modeling it as a stream.
