# Caching

Caching means storing the result of an expensive operation so a repeated request for the same thing doesn't redo the work. The core design questions are always: what's the cache key, when does an entry get invalidated, and what happens on a miss.

## In the nested app

`lib/widgets/search_delegate.dart`'s `TreeSearchDelegate` caches the in-flight search `Future` itself, keyed by the query string, to avoid re-querying SQLite on every keystroke-triggered rebuild:

```dart
class TreeSearchDelegate extends SearchDelegate<TreeNode?> {
  Future<List<TreeNode>>? _cachedFuture;
  String? _cachedQuery;

  Future<List<TreeNode>> _getResultsFuture(NestedTreeNotifier notifier, String q) {
    if (q == _cachedQuery && _cachedFuture != null) return _cachedFuture!;
    _cachedQuery = q;
    _cachedFuture = notifier.searchTree(q);
    return _cachedFuture!;
  }
  ...
}
```

`SearchDelegate`'s `buildSuggestions`/`buildResults` can be called by the framework multiple times for the *same* query (rebuilds, focus changes) — without this cache, each rebuild would call `_getResultsFuture` → `notifier.searchTree(q)` → a fresh `SELECT ... LIKE` query, even though nothing about the query changed. Caching the `Future` itself (not just the eventual result) also means a `FutureBuilder` watching it won't flash back to a loading state on a rebuild that happens while the original query is still in flight — it's handed the same still-pending `Future` instead of starting a new one.

The invalidation rule here is intentionally simple: the cache holds exactly one entry, and it's invalidated the instant the query string changes (`q == _cachedQuery` is the only check). That's the right amount of caching for this use case — there's no need to remember results for five different past queries when a search UI only ever shows results for the current query box contents.

## Key takeaways

- Cache the smallest thing that actually needs it — here, one `Future` for the *current* query, not a general-purpose results cache.
- Caching a `Future` (not just its resolved value) avoids re-triggering the underlying async work on rebuilds that happen while it's still pending — `FutureBuilder` just re-attaches to the same instance.
- Invalidation strategy should match how the data is actually used: single-entry, keyed-by-current-query caching is sufficient here because old results are never shown again once the query changes; a cache serving multiple concurrent consumers (e.g., folder metadata reused across several screens) would need a real eviction policy instead.
- This app's [[SQLite]] queries themselves aren't cached beyond this — `NestedTreeNotifier._load()` re-queries on every navigation, which is correct here since the underlying data can change between visits (imports, syncs, edits) and staleness would be a worse bug than the extra query cost.
