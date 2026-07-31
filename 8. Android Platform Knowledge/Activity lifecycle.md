# Activity lifecycle

An `Activity`'s lifecycle (`onCreate` → `onStart` → `onResume` → ... → `onPause` → `onStop` → `onDestroy`) governs a single screen's existence independent of whatever Flutter widget lifecycle sits on top of it (see [[Widget lifecycle]] for that separate, Dart-side concept). For a Flutter app, `FlutterActivity` handles almost all of this for you — but this app's Android-native activities (used for widget interactions, not Flutter screens) show the raw lifecycle callbacks doing real work.

## In the nested app

**`MainActivity`** is the one `FlutterActivity` in the app — its own lifecycle overrides are narrow and specific, just enough to capture "which folder should the app open to" when launched via a widget tap:

```kotlin
class MainActivity : FlutterActivity() {
    companion object {
        private var pendingFolderId: Long? = null
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        extractFolderId(intent)?.let { pendingFolderId = it }
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        setIntent(intent)
        extractFolderId(intent)?.let {
            pendingFolderId = it
            widgetNavChannel?.invokeMethod("onOpenFolder", it)
        }
    }
}
```

`onCreate` (cold start — the process didn't exist yet) just stashes the folder id in a static field for Dart to pull later via `getInitialFolderId` (see [[MethodChannel]]); `onNewIntent` (hot start — the activity's already running) does the same *and* pushes it immediately across the channel, since there's a live Dart side already listening. That split exists precisely because these are two different points in the lifecycle with different capabilities — at `onCreate` time, the Flutter engine and its method channels aren't wired up yet.

**`AddTaskActivity`** and **`ShareTargetActivity`** are plain (non-Flutter) `Activity` subclasses — "trampoline" dialogs that appear, do one small job, and finish, without ever touching the Flutter engine at all:

```kotlin
class AddTaskActivity : Activity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_add_task)
        window.setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_STATE_ALWAYS_VISIBLE)
        ...
        nameField.requestFocus()
        findViewById<Button>(R.id.add_save).setOnClickListener {
            ...
            NodeDbHelper(this).insertTask(parentId, name, priority)
            finish()
        }
    }

    override fun onWindowFocusChanged(hasFocus: Boolean) {
        super.onWindowFocusChanged(hasFocus)
        if (hasFocus) {
            nameField.requestFocus()
            val imm = getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
            imm.showSoftInput(nameField, InputMethodManager.SHOW_IMPLICIT)
        }
    }
}
```

`onWindowFocusChanged` (not part of the create/start/resume sequence, but a related lifecycle-adjacent callback) is used here specifically because `requestFocus()` called too early — before the window actually has focus — doesn't reliably bring up the soft keyboard; waiting for `hasFocus == true` is the standard workaround for "show the keyboard immediately when this dialog-style activity appears."

Both trampoline activities are deliberately minimal in the manifest — `android:excludeFromRecents="true"` and `android:taskAffinity=""` (see [[AndroidManifest]]) keep them from cluttering the recent-apps list or merging into the main app's task, appropriate for something meant to flash briefly and disappear.

## Key takeaways

- `onCreate` runs once per activity instance; `onNewIntent` is what handles a *new* intent arriving at an already-running instance (relevant whenever `launchMode` isn't the default) — see [[Intents]] for why `MainActivity` needs both.
- A `FlutterActivity` still has a real native Android lifecycle underneath the Flutter engine — this app's `MainActivity` overrides are proof that native lifecycle hooks and Flutter's own widget lifecycle coexist, not compete.
- Not every native `Activity` in a Flutter app needs to touch the Flutter engine at all — `AddTaskActivity`/`ShareTargetActivity` are complete, working native UI that never starts a `FlutterEngine`, appropriate for something that needs to be fast and lightweight (a widget-triggered quick-add dialog).
- `onWindowFocusChanged` is the reliable point to force-show the keyboard on a freshly shown activity — `requestFocus()` alone in `onCreate` is not.
