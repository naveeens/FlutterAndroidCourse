# File I/O

"File I/O" here means reading/writing files on the device's filesystem directly (`dart:io`'s `File`, `Directory`), as opposed to going through a structured store like [[SQLite]] or a key-value store like [[SharedPreferences]]. Typical uses: exporting user data to a file, caching downloaded assets, reading configuration bundled with the app.

## Why the nested app mostly avoids it — and where it comes close

This app deliberately doesn't touch `dart:io`'s `File` API anywhere in `lib/` — everything either goes through SQLite (the actual tree data) or SharedPreferences (settings). Even its two file-shaped features work entirely **in memory**, which is worth understanding as a design choice, not an oversight.

**Export**, in `lib/folder_screen.dart`'s `_exportSelected`, builds a JSON string in memory and hands it to `share_plus` rather than writing a file to disk first:

```dart
Future<void> _exportSelected(NestedTreeNotifier notifier) async {
  ...
  final exportJson = await notifier.generateExportJson(selectedNodes);
  await SharePlus.instance.share(ShareParams(text: exportJson, subject: 'Exported JSON'));
  ...
}
```

`SharePlus.instance.share(...)` hands the string straight to the OS share sheet — the user picks where it goes (a messaging app, email, a notes app, a real file via "Save to Files"), and the OS/receiving app deals with turning it into a file if needed. The app itself never manages a file path, cleans up a temp file, or worries about storage permissions.

**Import**, in `lib/services/github_import_service.dart`, downloads a zip via HTTP and decodes it entirely in memory using `package:archive`:

```dart
final zipResponse = await http.get(zipUri);
final archive = ZipDecoder().decodeBytes(zipResponse.bodyBytes);
for (final entry in archive) {
  if (!entry.isFile) continue;
  ...
  content: utf8.decode(entry.content as List<int>, allowMalformed: true),
}
```

`ZipDecoder().decodeBytes(...)` operates on the raw byte array from the HTTP response — no temp file is ever written to disk, unzipped, and cleaned up. `entry.content` for each file inside the archive is also just bytes, decoded straight to a Dart `String`.

## When this app *would* need real File I/O

If a future version let users export to a chosen folder on device storage (rather than through the share sheet) or cache downloaded repo content between sessions, that's when `path_provider`'s `getApplicationDocumentsDirectory()` plus `dart:io`'s `File(path).writeAsString(...)` would become necessary — along with the platform permission handling that comes with touching shared storage directly (see [[Storage Access Framework]], [[Permissions]]).

## Key takeaways

- In-memory byte/string processing (`http` response bytes → `archive` → decoded strings, or a generated string → `share_plus`) sidesteps file I/O entirely when you don't actually need a persistent file — no cleanup, no path management, no storage permissions to request.
- Reach for real `File`/`Directory` APIs when data genuinely needs to persist as a file the user or another app can find later — a one-shot export via the OS share sheet, as this app does, usually doesn't.
- `share_plus`'s `ShareParams(text: ...)` sharing a string vs. sharing an actual `XFile` are different code paths — this app only ever needed the former.
