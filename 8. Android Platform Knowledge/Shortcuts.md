# Shortcuts

Android app shortcuts (long-press an app icon on the home screen) let an app expose a handful of quick actions — "static" ones declared once in XML, or "dynamic"/"pinned" ones an app publishes and updates at runtime via `ShortcutManager` — as a faster way into a specific piece of functionality than opening the app and navigating there manually.

## Not used in the nested app

This app declares no shortcuts — no `res/xml/shortcuts.xml`, no `ShortcutManagerCompat` calls anywhere in the Kotlin code. Long-pressing the app's launcher icon today shows only the OS default (uninstall/app-info), nothing custom.

That's a real, notable gap given what this app already has: the **home-screen widget already solves a large chunk of what shortcuts are typically used for** — quick task entry (the widget's add button → `AddTaskActivity`, a trampoline dialog that opens, adds a task, and closes — see [[Activity lifecycle]]) and quick navigation into a specific pinned folder (the widget's prev/next arrows and header tap). A user who wants "add a task without fully opening the app" already has that, just via a widget rather than a long-press shortcut.

## Where shortcuts would still add value

Static or dynamic shortcuts are still worth adding on top of the widget, because they're reachable without the user having placed a widget on their home screen at all — shortcuts work from the launcher icon itself, which every user has by definition. A natural set for this app: "Add Task," "Search," and dynamic shortcuts for the user's most-recently-visited pinned folders (mirroring `getPinnedFoldersOrdered()` in `NodeDbHelper.kt`, which the widget already uses for its own folder-paging).

Roughly, a dynamic shortcut updated whenever pinned folders change:

```kotlin
val shortcut = ShortcutInfoCompat.Builder(context, "folder_$folderId")
    .setShortLabel(folder.name)
    .setIcon(IconCompat.createWithResource(context, R.drawable.ic_folder))
    .setIntent(Intent(context, MainActivity::class.java).apply {
        putExtra(NestedWidgetProvider.EXTRA_PARENT_ID, folderId)
        action = Intent.ACTION_VIEW
    })
    .build()
ShortcutManagerCompat.pushDynamicShortcut(context, shortcut)
```

Note this would reuse the *exact* same deep-link mechanism the widget already established — `MainActivity` reading `EXTRA_PARENT_ID` out of the launch intent (see [[Intents]], [[Activity lifecycle]]) doesn't care whether that intent came from a widget tap or a shortcut tap; the entry point is already shortcut-ready without any `MainActivity` changes.

## Key takeaways

- App shortcuts and home-screen widgets solve overlapping but distinct problems — a widget needs to be explicitly placed and takes home-screen space; a shortcut is always available from the launcher icon with zero setup.
- This app's existing widget-tap deep-link mechanism (`EXTRA_PARENT_ID` read in `MainActivity`) would need no changes to also serve as a shortcut's target intent — the plumbing already exists, only the shortcut declarations themselves are missing.
- Dynamic shortcuts (updated at runtime, e.g. to reflect pinned folders) would parallel data this app already tracks (`getPinnedFoldersOrdered` in `NodeDbHelper.kt`) rather than needing new state.
