# Authentication

Authenticating a network request usually means attaching credentials — an API key, a bearer token, a session cookie — so the server knows who's asking, either to grant access to private data or to raise/attribute rate limits.

## Not used in the nested app — and why that's a real, deliberate constraint

`GithubImportService` (`lib/services/github_import_service.dart`) makes every request unauthenticated:

```dart
final response = await http.get(uri, headers: {'Accept': 'application/vnd.github+json'});
```

No `Authorization` header, anywhere. This is why the service's docstring is explicit that it only supports **public** repos — GitHub's API simply returns 404 for a private repo you're not authenticated for, which is exactly what `_fetchDefaultBranch`'s `if (response.statusCode != 200) throw Exception('Repo not found or not public.')` surfaces to the user. Unauthenticated access is also why the service is careful to make only one REST API call per import (see [[REST APIs]]) — GitHub's unauthenticated rate limit (60 requests/hour per IP) is far more restrictive than the 5,000/hour an authenticated request gets, so authentication would also *raise* the app's practical ceiling on API usage, not just unlock private repos.

## What adding it would require

Supporting private repos would mean sending a GitHub personal access token or OAuth token as a bearer credential:

```dart
final response = await http.get(
  uri,
  headers: {
    'Accept': 'application/vnd.github+json',
    'Authorization': 'Bearer $githubToken',
  },
);
```

The harder part isn't the header — it's everywhere else authentication touches an app: where `githubToken` comes from (an OAuth flow, or a user-pasted personal access token), and how it's stored between launches. Given this app already keeps user data in plain SQLite/SharedPreferences (see [[Encryption]]), a credential like this specifically would need to go through `flutter_secure_storage` instead, not the same unencrypted `SharedPreferences` the app currently uses for its theme key — credentials are exactly the category of data where that distinction matters.

## Key takeaways

- Unauthenticated access is a legitimate, deliberate choice when it fits the feature (public-only import here) — it trades away private-data access and a higher rate limit for zero credential-management complexity.
- Whenever an app *does* add authentication, the token/credential itself needs secure storage (see [[Encryption]]) — it should never sit alongside plain app settings in unencrypted `SharedPreferences`.
- A `401`/`403` from an API almost always means "missing or invalid credential," distinct from this app's current `404` ("not found or not public") — a future authenticated version of `GithubImportService` would need to distinguish those cases in its error handling.
- See [[JWT]] for one common concrete credential format, and [[REST APIs]] for how the app's single unauthenticated call fits into its broader network usage.
