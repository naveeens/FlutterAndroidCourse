# Storage Access Framework

The Storage Access Framework (SAF) is Android's system-mediated way of letting an app read/write files the user explicitly picks (or a folder the user grants ongoing access to), via `Intent.ACTION_OPEN_DOCUMENT`/`ACTION_CREATE_DOCUMENT`/`ACTION_OPEN_DOCUMENT_TREE` — the system shows a file/folder picker UI, and the app receives a `content://` URI it can read/write without needing broad storage permissions. It's the modern replacement for apps requesting blanket external-storage access.

## Not used in the nested app — export/import sidesteps it entirely

This app never opens a system file picker. Its two file-shaped features — export and import — both route around needing one at all (see [[File I-O]] for the full picture):

**Export** hands a generated string straight to the OS share sheet rather than writing to a user-chosen file location:

```dart
final exportJson = await notifier.generateExportJson(selectedNodes);
await SharePlus.instance.share(ShareParams(text: exportJson, subject: 'Exported JSON'));
```

The user picks *where* it ends up (a messaging app, "Save to Drive," a file manager) through the share sheet — which is itself effectively a doorway into SAF-like pickers on the receiving end, but this app's own code never talks to SAF directly.

**Import** happens two ways, neither touching SAF: pasting JSON text directly into a text field (`_showJsonImportSheet` in `folder_screen.dart`), or fetching a GitHub repo over HTTP (`GithubImportService` — see [[Downloads]]). Neither reads a file the user picked from local device storage.

## Where this app would need it

If a future version let a user import a JSON export *file* (rather than pasting its text) or export directly to a chosen folder instead of going through the share sheet, that's exactly what SAF is for:

```dart
// via package:file_picker, which wraps SAF's ACTION_OPEN_DOCUMENT under the hood
final result = await FilePicker.platform.pickFiles(type: FileType.custom, allowedExtensions: ['json']);
if (result != null) {
  final content = await File(result.files.single.path!).readAsString();
  await notifier.importFromJson(content);
}
```

The key SAF property worth understanding even without in-app code to point to: the returned URI/path is scoped specifically to the file or folder the user picked — the app never gets broad filesystem access, and if the user picked a single file, the app can't wander to sibling files in the same folder without a fresh picker interaction. That's a meaningfully different (and more restrictive, by design) model than the legacy "grant `READ_EXTERNAL_STORAGE`, read anywhere" approach it replaced.

## Key takeaways

- SAF trades broad storage permission for per-file/per-folder, user-granted, system-mediated access — the current standard, privacy-respecting way to let a user pick a file for your app to read/write.
- This app avoids needing it entirely by using the OS share sheet for export and direct paste/network fetch for import — legitimate alternatives when you don't need arbitrary local file access.
- The natural trigger to adopt it: letting a user pick an existing file from their device (rather than pasting content or fetching from a URL) — something like importing a previously-exported JSON file directly.
