# Server-Sent Events

Server-Sent Events (SSE) is a one-way streaming protocol over plain HTTP — the server keeps a connection open and pushes a sequence of text events to the client over time, without the client needing to poll. It's simpler than a [[WebSockets]] connection (built on ordinary HTTP, one direction only, auto-reconnects per spec) and a good fit for things like live progress updates, notification feeds, or streaming an LLM's token-by-token response.

## Not used in the nested app

There's no long-lived, server-pushed data anywhere in this app. Its one network integration, `GithubImportService` (`lib/services/github_import_service.dart`), makes exactly two short-lived request/response HTTP calls (see [[HTTP requests]]) and is done — there's no ongoing connection to keep open, and nothing on the server side is pushing updates to the app unprompted. Every piece of "live" data the app displays (the folder tree, task counts, breadcrumbs) updates because of a local `notifyListeners()` call after a local SQLite write (see [[ChangeNotifier]]), not because anything arrived over the network.

## Where SSE would fit if this app added it

The clearest hypothetical: if `importFromGithub`/`syncGithubFolder` handled very large repos and the app wanted to show live per-file progress ("importing docs/setup.md... 34 of 210 files") from a backend doing the heavy lifting server-side, SSE is a natural fit — the client opens one connection and receives a stream of progress events until the import completes, rather than polling a status endpoint repeatedly. Roughly:

```dart
final client = http.Client();
final request = http.Request('GET', Uri.parse('https://api.example.com/import-progress/$jobId'));
final response = await client.send(request);

await for (final chunk in response.stream.transform(utf8.decoder)) {
  // parse "data: {...}\n\n" formatted SSE events out of chunk
}
```

That's meaningfully more infrastructure than this app's current design needs, though — the actual `importFromGithub` does its work entirely on-device (download zip, decode, write to SQLite; see [[Downloads]]), so there's no server-side job to report progress *from* in the first place. SSE only becomes relevant once there's a long-running *server-side* process to observe.

## Key takeaways

- SSE is one-directional (server → client) streaming over plain HTTP — simpler than WebSockets when you only need push updates, not bidirectional communication.
- It requires an actual long-running server-side process to stream progress from — this app's GitHub import does its work client-side, so there's nothing server-side to subscribe to.
- If this app grew a backend for larger/slower operations, SSE (for one-way progress/status feeds) is worth reaching for before WebSockets, which solves a strictly harder (bidirectional) problem than most progress-reporting use cases actually need.
