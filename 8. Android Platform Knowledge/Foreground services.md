# Foreground services

A foreground service is a component that keeps running with elevated priority (unlikely to be killed for memory) *while showing the user a persistent, ongoing notification* — the notification is the deal Android requires in exchange for letting your code keep running: the user always knows something is actively happening (music playback, an active GPS route, an ongoing file transfer/call).

## Not used in the nested app

No feature in this app runs long enough, or needs to keep running while the app is backgrounded, to justify one. Its longest-running operation — `GithubImportService.fetchMarkdownFiles` (download a repo zip, decode it; see [[Downloads]]) — completes in seconds and is triggered and awaited entirely within the foreground Flutter UI (`folder_screen.dart`'s import/sync sheets), never continuing after the user navigates away or backgrounds the app.

The voice dictation feature (`VoiceRecordButton` in `lib/widgets/note_editor_screen.dart`) is the closest this app comes to "ongoing background-ish work" — a `speech_to_text` recording session that can run for tens of seconds. But it's explicitly foreground-UI-bound: the mic button's state, the recording indicator, and the transcription all live inside the currently-open `NoteEditorScreen`, with no expectation (or mechanism) for it to keep listening after the user leaves that screen — closing the note or backgrounding the app would reasonably stop it, not something a foreground service's "keep going regardless" guarantee is meant for.

## What would actually need one

If this app added a feature like "keep recording a voice note even if you switch to another app," or "continuously watch a GitHub repo for changes and sync live," either would cross the line into needing a foreground service — because both require the OS to *not* kill the process while the app isn't visible, which is exactly the guarantee a plain background coroutine/thread doesn't have. Sketching what that would look like for a hypothetical "keep recording in background" feature:

```kotlin
class RecordingService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("Recording voice note").setSmallIcon(R.drawable.ic_mic)
            .setOngoing(true).build()
        startForeground(NOTIFICATION_ID, notification)
        // ... actual recording logic
        return START_STICKY
    }
}
```

Note the mandatory, non-optional notification (`setOngoing(true)`, `startForeground(...)`) — this is what distinguishes a foreground service from ordinary background work; the user must always be able to see, and stop, whatever's running on their behalf. See [[Notification channels]] for the notification-channel setup a real implementation of this would additionally require (mandatory on Android 8+).

## Key takeaways

- A foreground service trades "the OS won't casually kill this" for a mandatory persistent notification — appropriate only for work the user should always be visibly aware is ongoing.
- This app's longest operations are short, foreground-UI-bound, and don't need to survive the user navigating away — none of its current features cross the threshold that would justify one.
- The clear trigger to add one: a feature that must keep running *specifically while the app is backgrounded or the screen is off* — this app has no such feature today.
