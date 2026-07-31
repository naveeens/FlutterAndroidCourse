# HTTP requests

`package:http` is Dart's standard, low-ceremony HTTP client — `http.get(uri)`, `http.post(uri, body: ...)`, each returning a `Future<http.Response>` with `.statusCode`, `.body`, `.bodyBytes`. It sits at a lower level than a full REST client wrapper: you build `Uri`s, check status codes, and decode bodies yourself, which is exactly enough for an app that only makes a couple of read-only calls.

## In the nested app

`lib/services/github_import_service.dart` makes the app's only two HTTP calls, both plain `GET` requests against public, unauthenticated endpoints:

```dart
static Future<String> _fetchDefaultBranch(String owner, String repo) async {
  final uri = Uri.parse('https://api.github.com/repos/$owner/$repo');
  final response = await http.get(uri, headers: {'Accept': 'application/vnd.github+json'});
  if (response.statusCode != 200) {
    throw Exception('Repo not found or not public.');
  }
  final decoded = jsonDecode(response.body) as Map<String, dynamic>;
  return decoded['default_branch'] as String? ?? 'main';
}
```

and, for the actual repo contents:

```dart
final zipUri = Uri.parse('https://codeload.github.com/$owner/$repo/zip/refs/heads/$branch');
final zipResponse = await http.get(zipUri);
if (zipResponse.statusCode != 200) {
  throw Exception('Could not download repo (${zipResponse.statusCode}).');
}
final archive = ZipDecoder().decodeBytes(zipResponse.bodyBytes);
```

Two habits worth internalizing from this code: always build the URL with `Uri.parse`/`Uri(...)` rather than string-concatenating query parameters (avoids encoding bugs), and always check `statusCode` explicitly before trusting the body — `http.get` does *not* throw on a 404 or 500, it returns a normal `Response` with that status code, so "check `statusCode != 200`, then throw a meaningful exception" is on you, as both methods here do.

`.body` (a decoded `String`, used for the JSON API response) versus `.bodyBytes` (raw `Uint8List`, used for the zip download) is the other distinction to notice — text responses want `.body` for `jsonDecode`, binary responses (here, a zip archive) need the untouched bytes via `.bodyBytes`.

## Key takeaways

- `http.get`/`http.post` never throw on non-2xx responses — checking `response.statusCode` yourself (as both methods above do) is required, not optional.
- `.body` for text/JSON, `.bodyBytes` for binary payloads — picking the wrong one either mangles binary data through string decoding or wastes a decode you don't need.
- `Uri.parse` for a fixed URL you're building from known-safe parts; `Uri(scheme: ..., host: ..., queryParameters: {...})` when parameters need proper encoding — this app's URLs are simple enough for `Uri.parse` with string interpolation, but that's a size call, not a rule to copy blindly for anything with user-supplied query values.
- See [[REST APIs]] for how these two calls compose into one higher-level operation, and [[Downloads]] for the zip-fetching side specifically.
