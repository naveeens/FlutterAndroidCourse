# Data migration

A schema migration is code that transforms an existing database from one version to the next — adding columns, changing types, backfilling data — without losing what's already stored. `sqflite`'s `openDatabase` takes a `version` number and an `onUpgrade(db, oldVersion, newVersion)` callback that runs automatically when a user's installed app opens a database file created by an older schema version.

## In the nested app

`lib/db/database_helper.dart` is currently on schema version 9, and its `_onUpgrade` shows two real migrations that shipped to users who already had data on their device:

```dart
static const _dbVersion = 9;

Future<void> _onUpgrade(Database db, int oldVersion, int newVersion) async {
  if (oldVersion < 8) {
    await db.execute('ALTER TABLE nodes ADD COLUMN is_pinned INTEGER NOT NULL DEFAULT 0');
  }
  if (oldVersion < 9) {
    await db.execute('ALTER TABLE nodes ADD COLUMN repo_url TEXT');
  }
}
```

Two details here matter for writing migrations correctly:

1. **Each `if` checks `oldVersion`, not `newVersion == X`.** A user could be upgrading from version 6 straight to 9 (skipped several app updates) — the `if (oldVersion < 8)` / `if (oldVersion < 9)` structure means *both* migrations run in sequence for that user, in order, rather than only running whichever single migration matches their exact starting version.
2. **`DEFAULT 0` on the new `is_pinned` column** means every existing row gets a sensible default value instantly, without needing a separate backfill `UPDATE` statement — SQLite fills it in as part of the `ALTER TABLE`.

Compare this to `_onCreate`, which only runs for a device with *no* existing database (a fresh install) — it creates the schema at its current, final shape (already including `is_pinned` and `repo_url` in the `CREATE TABLE` statement) rather than creating an old version and migrating it forward. That's deliberate: `_onCreate` and `_onUpgrade` are two separate paths to the same end state, and keeping `_onCreate` current avoids a fresh install replaying nine versions' worth of migrations for no reason.

## Key takeaways

- Bump `_dbVersion` every time the schema changes, and add exactly one new `if (oldVersion < N)` block per version bump — never edit or remove an old migration block once it's shipped, since real user devices may still be on that old version.
- Branch migrations on `oldVersion`, not on checking the current schema state — this is what makes multi-version jumps (skipping several app updates) apply every needed migration in order.
- Give new columns a `DEFAULT` (as `is_pinned INTEGER NOT NULL DEFAULT 0` does) whenever existing rows need a sane value — otherwise you need an explicit `UPDATE` to backfill before any `NOT NULL` constraint can be added safely.
- Keep `_onCreate`'s schema in sync with the *cumulative* result of all migrations — a fresh install should never have to run `_onUpgrade` at all.
