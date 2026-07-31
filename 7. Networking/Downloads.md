# Downloads

Downloading a file in Flutter, at the simplest level, is just an HTTP `GET` whose response body you treat as bytes instead of text — no special "download" API is required for something that fits in memory. For very large files or ones needing progress reporting/resumability, you'd stream the response instead (`http.Client().send(request)` and read the byte stream incrementally) rather than awaiting the whole body at once.

## In the nested app

`GithubImportService.fetchMarkdownFiles` (`lib/services/github_import_service.dart`) downloads an entire GitHub repository as a zip archive — using the same URL structure GitHub's own "Download ZIP" button and `git clone` rely on (`codeload.github.com`), not the REST API:

```dart
final zipUri = Uri.parse('https://codeload.github.com/$owner/$repo/zip/refs/heads/$branch');
final zipResponse = await http.get(zipUri);
if (zipResponse.statusCode != 200) {
  throw Exception('Could not download repo (${zipResponse.statusCode}).');
}

final archive = ZipDecoder().decodeBytes(zipResponse.bodyBytes);
```

This is the simplest possible download shape: one `http.get`, the whole response awaited into memory, then handed straight to `ZipDecoder().decodeBytes(...)` — no temp file ever touches disk (see [[File I-O]]), no progress callback, no resumability. That's a reasonable choice here because the payload is a text-only markdown repo, which in practice is small (typically well under a few MB) — awaiting the full response is simpler and, for this size of payload, not meaningfully slower than streaming it.

Once decoded, each archive entry's path is cleaned up before use:

```dart
for (final entry in archive) {
  if (!entry.isFile) continue;
  if (!entry.name.toLowerCase().endsWith('.md')) continue;

  final parts = entry.name.split('/');
  final relativeParts = parts.length > 1 ? parts.sublist(1) : parts;
  if (relativeParts.isEmpty || relativeParts.last.isEmpty) continue;

  files.add(GithubMarkdownFile(
    pathSegments: relativeParts,
    content: utf8.decode(entry.content as List<int>, allowMalformed: true),
  ));
}
```

`parts.sublist(1)` strips the zip's own top-level `repo-branch/` wrapper folder that GitHub's zip format always includes — a small but necessary bit of "downloaded archive" housekeeping that has nothing to do with the HTTP layer itself but is easy to forget.

## What this would look like for a large binary download

If this app ever downloaded something large or binary (say, an attachment feature), the same `http.get`-awaits-everything approach would be the wrong call — you'd want `http.Client().send(http.Request('GET', uri))` and read `response.stream` incrementally, writing chunks to a `File` (via `dart:io`) as they arrive, both to avoid holding the whole payload in memory and to support showing real download progress from `response.contentLength` vs. bytes received so far.

## Key takeaways

- `http.get(...).bodyBytes` is sufficient for downloads that comfortably fit in memory — no need to reach for streaming APIs until payload size or the need for progress/resumability actually demands it.
- Bulk downloads (a zip, here) can be a legitimate alternative to many small REST API calls — see [[REST APIs]] for why this app chose the zip route over listing files individually.
- Archive formats often wrap content in their own top-level folder (GitHub's zips do) — that's a downloaded-archive detail to handle, distinct from anything HTTP-specific.
