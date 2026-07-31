# Multipart uploads

A multipart/form-data request is how HTTP sends files (plus optionally regular fields) in a single request — the body is split into named parts, each with its own headers, letting a server receive binary file data alongside form fields like a caption or filename. In `package:http`, this is `http.MultipartRequest` instead of the plain `http.get`/`http.post` used for JSON bodies.

## Not used in the nested app

Every network call in this app is a `GET` — `lib/services/github_import_service.dart` only ever *downloads* (repo metadata, then a zip archive; see [[Downloads]]). There's no feature that sends a file *to* a server, so there's no multipart upload code anywhere in this codebase.

## What it would look like if this app added one

The natural feature that would need it: exporting a folder as a file attachment to some backend, or uploading an image attached to a task note (this app currently has no image-attachment feature — notes are markdown text only, rendered via `flutter_markdown_plus` in `lib/widgets/note_editor_screen.dart`). If task notes ever supported image attachments backed by a remote server, the upload would look like:

```dart
Future<void> uploadAttachment(File imageFile, int taskId) async {
  final uri = Uri.parse('https://api.example.com/attachments');
  final request = http.MultipartRequest('POST', uri)
    ..fields['taskId'] = taskId.toString()
    ..files.add(await http.MultipartFile.fromPath('image', imageFile.path));

  final streamedResponse = await request.send();
  final response = await http.Response.fromStream(streamedResponse);
  if (response.statusCode != 200) {
    throw Exception('Upload failed (${response.statusCode}).');
  }
}
```

The key differences from this app's actual `http.get` calls: `MultipartRequest` builds the request incrementally (`.fields` for form values, `.files` for attached files) rather than a single call, `.send()` returns a `StreamedResponse` (since a multipart body can be large and is sent as a stream) rather than the immediately-awaited `Response` that `http.get` gives you, and `http.Response.fromStream(...)` is how you collect that streamed response back into something with a familiar `.statusCode`/`.body` if you don't need to process it incrementally.

## Key takeaways

- Multipart is specifically for sending files (optionally with form fields) — nothing in this app's current feature set (read-only GitHub import, local SQLite storage) needs it, which is why it's absent.
- `http.MultipartRequest` + `.send()` (streamed) replaces `http.get`/`http.post` (immediately awaited) as soon as you're uploading a file, not just sending JSON.
- If this app added attachments, the upload would sit alongside — not replace — its existing DAOs (see [[SQLite]]): a task's local note stays in SQLite; only the attached file's bytes would need to leave the device.
