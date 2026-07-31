# AndroidManifest

`AndroidManifest.xml` declares everything the OS needs to know about your app *before* running any of your code: what components exist (activities, receivers, services), what permissions it needs, what it can be launched by (intent filters), and what other apps' components it's allowed to query. A Flutter app's manifest lives at `android/app/src/main/AndroidManifest.xml` and is exactly as real and load-bearing as it would be in a pure-native Android app — Flutter doesn't abstract this layer away.

## Walking the nested app's manifest

**Permissions**, declared once at the top level:

```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.RECORD_AUDIO"/>
```

`INTERNET` backs the GitHub import feature (see [[HTTP requests]]); `RECORD_AUDIO` backs the note editor's voice-to-text button — see [[Permissions]] for the runtime side of `RECORD_AUDIO` specifically, since a manifest permission alone isn't enough for a dangerous-level permission like microphone access.

**The main Flutter activity**:

```xml
<activity android:name=".MainActivity" android:exported="true" android:launchMode="singleTop"
    android:taskAffinity="" android:theme="@style/LaunchTheme"
    android:windowSoftInputMode="adjustResize">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
```

`MAIN`/`LAUNCHER` is what makes this the app's home-screen icon entry point. `launchMode="singleTop"` and `exported="true"` are what make widget-tap navigation work at all — see [[Activity lifecycle]] and [[Intents]] for why.

**Trampoline activities** for the share target and widget quick-add/edit dialogs, each deliberately scoped down:

```xml
<activity android:name=".widget.AddTaskActivity" android:exported="false"
    android:theme="@style/TrampolineDialogTheme" android:excludeFromRecents="true" android:taskAffinity="" />
```

`exported="false"` (only this app can start it), `excludeFromRecents="true"` (doesn't clutter the recent-apps switcher for something meant to flash briefly), `taskAffinity=""` (doesn't merge into the main app's task stack). `ShareTargetActivity`, by contrast, needs `exported="true"` — it has to be startable by *other* apps' share sheets, which is the entire point of it existing.

**The widget provider, receivers, and service** — see [[Widgets (App Widgets)]], [[Broadcast Receivers]] for what each does; the manifest is where they're wired to the system events that trigger them:

```xml
<receiver android:name=".widget.NestedWidgetProvider" android:exported="true">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data android:name="android.appwidget.provider" android:resource="@xml/nested_widget_info" />
</receiver>
<receiver android:name=".widget.WidgetRowActionReceiver" android:exported="false" />
<service android:name=".widget.NestedRemoteViewsService"
    android:permission="android.permission.BIND_REMOTEVIEWS" android:exported="false" />
```

The service's `android:permission="android.permission.BIND_REMOTEVIEWS"` is a system-enforced requirement — only the system (the launcher, binding on behalf of the widget host) is allowed to bind to a `RemoteViewsService`, which is what `exported="false"` plus this permission jointly guarantee.

**`<queries>`** — a newer (Android 11+) manifest block declaring which *other* apps' components this app is allowed to see/query, part of package-visibility restrictions introduced to limit apps enumerating what else is installed:

```xml
<queries>
    <intent><action android:name="android.intent.action.PROCESS_TEXT"/><data android:mimeType="text/plain"/></intent>
    <intent><action android:name="android.speech.RecognitionService" /></intent>
    <intent><action android:name="android.intent.action.VIEW" /><category android:name="android.intent.category.BROWSABLE" /><data android:scheme="https" /></intent>
</queries>
```

Without these declarations, the app couldn't reliably detect whether a speech-recognition service or a browser is available on the device — Android would hide that information by default under package visibility rules.

## Key takeaways

- Every component (`Activity`, `Receiver`, `Service`) an app uses must be declared in the manifest — Flutter's own `MainActivity` is no exception, and every native widget component in this app has its own explicit entry.
- `exported` controls whether other apps can start/trigger a component — `true` only when that's genuinely intended (a launcher entry point, a share target), `false` (the safer default) for anything meant to be internal.
- `<queries>` is required (Android 11+) to check for the presence of other apps'/services' components you don't control — this app declares it for text-processing, speech recognition, and browser availability.
- See [[Permissions]] for the distinction between a manifest-declared permission and the runtime prompt some permissions additionally require.
