# Encryption

Encrypting local storage protects data at rest — if a device is lost, rooted, or backed up somewhere insecure, an encrypted database/file can't be read without the key, whereas a plain SQLite `.db` file or `SharedPreferences` XML file can be opened and read directly by anyone with filesystem access (trivial on a rooted device or via a backup extraction tool).

## Not used in the nested app — and a reasonable call given what it stores

This app's database (`nested.db`, opened via plain `openDatabase` in `lib/db/database_helper.dart`) and its `SharedPreferences` values (theme key, accent color) are unencrypted on disk. That's a defensible choice here: the data is personal task/note organization, not credentials, financial data, or health information — the kind of content where "someone with physical access to an unlocked, rooted device could read your task list" is a low-severity risk relative to the complexity encryption would add.

Contrast that with what *would* need encryption if this app grew: the `syncGithubFolder`/`importFromGithub` flow in `lib/providers/tree_notifier.dart` currently only handles **public** repos — `GithubImportService.fetchMarkdownFiles` calls the GitHub API and codeload host with no authentication. If a future version added private-repo support, it would need a GitHub personal access token or OAuth credential stored on-device — and that absolutely should not sit in plain `SharedPreferences` or an unencrypted SQLite column next to task names.

## What that would look like

For a credential like a GitHub token, the standard Flutter approach is `flutter_secure_storage`, which uses the platform's real secure storage (Android Keystore-backed `EncryptedSharedPreferences`, iOS Keychain) rather than hand-rolled encryption:

```dart
final storage = FlutterSecureStorage();
await storage.write(key: 'github_token', value: token);
final token = await storage.read(key: 'github_token');
```

For encrypting the SQLite database itself (if it started holding sensitive content), `sqflite_sqlcipher` is the drop-in replacement for `sqflite` — same API surface as what `database_helper.dart` already uses, but the `.db` file on disk is encrypted with a passphrase you supply at `openDatabase` time.

## Key takeaways

- Encryption is a cost (complexity, key management, sometimes performance) that should match the sensitivity of what's actually stored — this app's plain SQLite/SharedPreferences setup is appropriate for its current data, not an oversight.
- Credentials/tokens (API keys, auth tokens) are the clear line where plain storage stops being acceptable — `flutter_secure_storage` is the standard tool the moment an app like this one adds anything requiring authentication (e.g., private GitHub repo import).
- `sqflite_sqlcipher` swaps in for `sqflite` with minimal API changes if bulk data (not just a token) ever needed at-rest encryption — this app's existing `DatabaseHelper`/DAO structure would barely need to change to adopt it.
