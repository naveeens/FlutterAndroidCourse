# WebSockets

A WebSocket is a persistent, bidirectional connection between client and server — after an initial HTTP handshake, both sides can send messages to each other at any time, with none of the per-message overhead of opening a new HTTP request each time. It's the right tool for genuinely two-way, low-latency communication: chat, live cursors/collaboration, real-time multiplayer state.

## Not used in the nested app

Nothing in this app needs bidirectional, persistent communication. `GithubImportService`'s two calls (see [[HTTP requests]]) are each a single request-response round trip, done — there's no reason to hold a connection open, and the app never needs to *push* anything to a server unprompted (it only ever reads from GitHub). Compare this to [[Streams]], which the app *does* use extensively (`WidgetNavService`'s `StreamController`) — that's Dart's in-process event stream abstraction, unrelated to network protocols; it just happens to share the word "stream."

## Where WebSockets would become relevant

If this app grew real-time multi-device sync — you edit a task on your phone, and it appears live on a tablet signed into the same account, without either device polling — a WebSocket connection to a sync server is the natural mechanism. Sketching the shape (nothing like this exists in the app today):

```dart
final channel = WebSocketChannel.connect(Uri.parse('wss://api.example.com/sync'));

channel.stream.listen((message) {
  final event = jsonDecode(message as String);
  // apply a remote change to local SQLite, then notifyListeners()
});

channel.sink.add(jsonEncode({'type': 'taskUpdated', 'taskId': 42, 'name': 'New name'}));
```

Note how naturally this would slot into the app's existing architecture: `channel.stream.listen(...)` would look almost identical to how `WidgetNavService` already subscribes to its own `Stream<int>` in `folder_screen.dart`'s `initState` — and a remote update arriving would still end up calling into `NestedTreeNotifier`'s existing mutation methods (or new ones alongside them) followed by `notifyListeners()`, exactly the same flow a local edit already goes through. The WebSocket would be a new *source* of changes, not a different way of propagating them through the UI.

## Key takeaways

- WebSockets solve genuine two-way, persistent communication — this app's single request-response GitHub calls don't need that, and using a WebSocket for a one-shot fetch would be unnecessary complexity.
- Don't confuse Dart's `Stream`/`StreamController` (an in-process async event abstraction, used throughout this app — see [[Streams]]) with the WebSocket network protocol — the former is a language feature; the latter is a specific transport you'd use *inside* a stream-based consumption pattern if you added real-time networking.
- If a real-time sync feature ever landed in this app, `WebSocketChannel.connect(...).stream.listen(...)` would integrate the same way `WidgetNavService`'s existing stream does — feeding into `NestedTreeNotifier`'s mutation methods and `notifyListeners()`, not replacing that architecture.
