# REST APIs

A REST API exposes resources at URLs, manipulated via HTTP methods (`GET` to read, `POST` to create, etc.) with structured (usually JSON) request/response bodies. Working with one from a Flutter app is mostly: build the right URL, parse the JSON response into your own model, handle non-success status codes explicitly.

## In the nested app

`lib/services/github_import_service.dart` talks to exactly one REST endpoint — GitHub's public repo API — to answer one question: "what's this repo's default branch?"

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

This is a deliberately minimal REST integration, and the comment in the source explains why: *"No GitHub REST API calls beyond one cheap, unauthenticated lookup of the repo's default branch, so this stays well clear of API rate limits regardless of repo size."* Unauthenticated GitHub API requests are rate-limited per IP (60/hour) — rather than using the REST API to list repo contents file-by-file (which for a large repo could be dozens of calls, quickly exhausting that limit), the app makes exactly one REST call to learn the branch name, then switches to `codeload.github.com` (a bulk zip download, not the REST API — see [[Downloads]]) to actually fetch the repo's contents in a single request regardless of size.

That's a real architectural lesson: knowing a REST API's rate limits and shape should inform *whether* you use it for a given piece of work, not just how you parse its responses. `parseRepoUrl`, right above `_fetchDefaultBranch` in the same file, extracts `owner`/`repo` from a pasted URL via regex before either network call happens — validating and shaping the request's inputs locally, so a malformed URL never even reaches the network:

```dart
static GithubRepoRef parseRepoUrl(String repoUrl) {
  final match = _repoUrlPattern.firstMatch(repoUrl.trim());
  if (match == null) {
    throw const FormatException('Not a GitHub repo link (expected github.com/owner/repo).');
  }
  return GithubRepoRef(owner: match.group(1)!, repo: match.group(2)!);
}
```

## Key takeaways

- Model the API's actual shape with a small typed class (`GithubRepoRef` here) instead of passing raw strings/maps around — `ref.owner`, `ref.repo`, `ref.canonicalUrl` are clearer call sites than re-parsing a URL each time they're needed.
- Rate limits and per-call cost should shape *which* endpoint you call, not just how you handle its response — this app deliberately avoids the REST API's per-file listing in favor of one bulk zip download.
- Validate/parse client-side before making the network call wherever possible (`parseRepoUrl` throwing on a bad URL before any `http.get` happens) — fail fast without spending a network round-trip on an input you already know is invalid.
- See [[Authentication]] for what this integration would need to add to support private repos, which the app currently doesn't.
