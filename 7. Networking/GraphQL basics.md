# GraphQL basics

GraphQL is a query language for APIs where the *client* specifies exactly which fields it wants, in a single request, potentially across what would be several REST endpoints — you send a query document, the server returns JSON shaped exactly like the query. Its main appeal over REST: no over-fetching (getting fields you don't need) or under-fetching (needing several round-trips to assemble one screen's worth of data).

## Not used in the nested app — and REST was the simpler, correct call

`GithubImportService` talks to GitHub's REST API (`api.github.com`), not GitHub's GraphQL API (`api.github.com/graphql`), which also exists and could answer the same "what's this repo's default branch" question. For a service making exactly one simple, single-field lookup per import (see [[REST APIs]]), GraphQL's core benefit — avoiding over/under-fetching across a *complex* data need — doesn't apply: there's only one field being fetched (`default_branch`), so there's nothing to over-fetch, and a plain REST `GET` is strictly simpler to write, read, and debug than constructing a GraphQL query document and a client to send it.

## What the same call would look like in GraphQL, for comparison

```dart
const query = r'''
  query($owner: String!, $name: String!) {
    repository(owner: $owner, name: $name) {
      defaultBranchRef { name }
    }
  }
''';

final response = await http.post(
  Uri.parse('https://api.github.com/graphql'),
  headers: {'Authorization': 'Bearer $token', 'Content-Type': 'application/json'},
  body: jsonEncode({'query': query, 'variables': {'owner': owner, 'name': name}}),
);
final data = jsonDecode(response.body)['data']['repository']['defaultBranchRef']['name'];
```

Notice this actually needs *more* code than `_fetchDefaultBranch`'s REST version for the identical result — a query document to write, and (a real practical point) GitHub's GraphQL API requires authentication even for public repos, which would immediately conflict with this service's deliberately unauthenticated design (see [[Authentication]]). GraphQL earns its complexity back once you're assembling many related fields from many resources in one screen — say, a repo's metadata, its recent commits, and its contributors all at once — which is exactly the kind of need this app's single-field lookup never has.

## Key takeaways

- GraphQL's value is proportional to how much your client needs to assemble from otherwise-scattered/nested REST resources — a single-field lookup like this app's `_fetchDefaultBranch` gets no benefit from it.
- Fewer, larger GraphQL queries trade off against REST's simplicity (no query language to construct, plain URLs, simpler caching semantics) — worth defaulting to REST until over/under-fetching becomes an actual, measured problem.
- Authentication requirements can differ meaningfully between an API's REST and GraphQL surfaces (as with GitHub's GraphQL API requiring auth where its REST equivalent for public data doesn't) — worth checking before assuming they're interchangeable for a given use case.
