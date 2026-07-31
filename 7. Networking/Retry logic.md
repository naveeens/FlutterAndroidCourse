# Retry logic

Networks fail transiently — a dropped packet, a momentary DNS hiccup, a server briefly overloaded. Retry logic re-attempts a failed request (usually with backoff — waiting longer between each successive retry, and often jitter to avoid many clients retrying in lockstep) instead of surfacing the very first failure straight to the user.

## Not used in the nested app — worth noticing as a real gap, not just an absence

`GithubImportService` (`lib/services/github_import_service.dart`) makes no retry attempt at all. Both of its HTTP calls fail on the first bad response and propagate straight up as an exception:

```dart
final response = await http.get(uri, headers: {'Accept': 'application/vnd.github+json'});
if (response.statusCode != 200) {
  throw Exception('Repo not found or not public.');
}
```

`folder_screen.dart`'s import/sync handlers catch that exception and show it in a `SnackBar` — a reasonable user experience for a *permanent* failure ("repo not found") but not distinguished at all from a *transient* one (a momentary network blip that a simple retry would have resolved). Unlike [[Offline-first]], where the absence of network dependency is a deliberate design win, this is a case where the app's simplicity is a genuine trade-off: a flaky connection means the user has to notice the error and manually tap "Import" again themselves.

## What adding retry logic here would look like

A minimal exponential-backoff retry wrapped around the existing calls, without changing anything else about the service:

```dart
Future<http.Response> _getWithRetry(Uri uri, {Map<String, String>? headers, int maxAttempts = 3}) async {
  for (var attempt = 1; ; attempt++) {
    try {
      final response = await http.get(uri, headers: headers);
      if (response.statusCode >= 500 && attempt < maxAttempts) {
        await Future.delayed(Duration(milliseconds: 300 * (1 << attempt))); // 600ms, 1200ms, ...
        continue;
      }
      return response;
    } on http.ClientException {
      if (attempt >= maxAttempts) rethrow;
      await Future.delayed(Duration(milliseconds: 300 * (1 << attempt)));
    }
  }
}
```

The important discipline this sketch shows: only retry on things retrying can actually fix — a `5xx` (server-side, possibly transient) or a connection-level exception, never a `4xx` like the `404` `_fetchDefaultBranch` throws for a private/nonexistent repo, since retrying "not found" just wastes time and attempts before showing the same, correct error the first attempt already had.

## Key takeaways

- Retry only transient failure classes (network exceptions, `5xx`) — retrying a `4xx` (bad request, not found, unauthorized) just delays showing the user an error that isn't going to change.
- Exponential backoff (each retry waits longer than the last) avoids hammering a server that's already struggling, and is standard even for a small number of attempts (2-3 is often enough for real transient blips).
- This app's current behavior — one attempt, then surface the error via `SnackBar` — is simple and honest, but a real, noticeable rough edge relative to a more resilient implementation; it's a reasonable next improvement if flaky-connection import failures turn out to be a common complaint.
