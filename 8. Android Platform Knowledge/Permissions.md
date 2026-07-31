# Permissions

Android permissions come in two practical tiers: "normal" permissions (like `INTERNET`) are granted automatically at install time just by being declared in the manifest — no runtime prompt. "Dangerous" permissions (microphone, camera, location, contacts, etc.) require *both* a manifest declaration *and* an explicit runtime request the user can approve or deny, introduced so users see exactly when a sensitive capability is actually being requested rather than blanket-approving everything at install.

## In the nested app

`AndroidManifest.xml` declares two permissions:

```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.RECORD_AUDIO"/>
```

`INTERNET` is a normal permission — the GitHub import feature (see [[HTTP requests]]) works the moment the app is installed, with no user-facing prompt ever shown for it.

`RECORD_AUDIO` is dangerous-tier, backing the note editor's voice dictation button (`VoiceRecordButton` in `lib/widgets/note_editor_screen.dart`). The manifest declaration alone isn't enough — the actual runtime prompt is triggered by the `speech_to_text` plugin the first time `SpeechToText.initialize()` is called, which this app deliberately defers until the user's *first tap* on the mic button rather than requesting it eagerly at app startup:

```dart
Future<void> _startRecording() async {
  if (_isStarting || _isStopping || _state != RecordState.idle) return;
  _isStarting = true;

  // Fix #4 — initialise (and show OS permission dialog) on first tap only.
  if (!_speechInitialized) {
    await _initSpeech();
  }

  if (!_speechAvailable) {
    if (mounted) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text(
          'Speech recognition not available. Please check microphone permissions.',
        )),
      );
    }
    _isStarting = false;
    return;
  }
  ...
}
```

`_speechAvailable` being `false` after `_initSpeech()` covers two real scenarios at once: the user denied the permission prompt, or the device genuinely has no speech-recognition engine — either way, this app surfaces one clear, actionable message instead of crashing or silently doing nothing. The comment (`// Fix #4`) marking this as a deliberate fix is worth noting too — requesting a dangerous permission immediately at launch (before the user has any context for why you need it) is a well-known bad pattern; deferring to the moment of actual use, as this app does, is the correct fix.

## Key takeaways

- Normal permissions (`INTERNET`, and most that don't touch sensitive hardware/data) need only a manifest entry; dangerous permissions (`RECORD_AUDIO`, camera, location, etc.) additionally require an explicit runtime request the user can deny.
- Request a dangerous permission at the moment its feature is actually used (as this app's mic button does), not proactively at launch — users are far more likely to grant a permission when its purpose is immediately obvious.
- Always handle the denied case explicitly and visibly (this app's `SnackBar`) — a dangerous permission being denied is a completely normal, expected outcome your code must handle gracefully, not an edge case to ignore.
- See [[AndroidManifest]] for where the declaration lives, and note that manifest declaration is necessary but not sufficient for anything dangerous-tier — the runtime request is a separate, required step handled here by the `speech_to_text` plugin.
