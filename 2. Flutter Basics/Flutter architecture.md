# Flutter architecture

Flutter apps are not "Dart talking to native widgets" — Flutter brings its own rendering engine (Skia/Impeller) and draws every pixel itself, on a canvas hosted inside a single native view. The three layers, roughly:

1. **Framework (Dart)** — widgets, the [[Element tree]], the [[Render tree]], gesture handling, everything you write and most of what you import from `package:flutter`.
2. **Engine (C++)** — Skia/Impeller for rendering, text layout, plugin architecture, the Dart runtime itself.
3. **Embedder (platform-specific)** — on Android, a thin Kotlin/Java shell (an `Activity` hosting a `FlutterView`) that gives the engine a surface to draw on and forwards platform events (lifecycle, input) into it.

This matters practically because Flutter's widgets — `Material`, `Cupertino`, all the layout primitives — are Dart-side implementations, not wrappers around native `UIView`/`android.view.View` equivalents. It's why a `Checkbox` looks and behaves identically on Android and iOS unless you deliberately reach for the Cupertino variants, and why Flutter's own rendering pipeline (not the OS's) is what you're reasoning about in [[Render tree]] and [[Frame rendering]].

## Where the nested app touches each layer

Almost everything in `lib/` lives in the framework layer — widgets, providers, DAOs. But the app also has a real embedder-layer component: **the Android home-screen widget** is not drawn by the Flutter engine at all. `android/app/src/main/kotlin/live/suture/nested/widget/NestedWidgetProvider.kt` and friends are pure native Android code using `RemoteViews`, running independently of whether the Flutter engine is even alive.

The bridge between the two is a `MethodChannel` — `lib/services/widget_bridge.dart` and `lib/services/widget_nav_service.dart` on the Dart side, `MainActivity.kt` on the native side. See [[Platform Channels]] and [[MethodChannel]] for how that crossing works. This is the clearest evidence in the app that "Flutter" and "the platform it runs on" are two separate architectures glued together deliberately, not one blended system.

`main.dart`'s use of `SystemChrome.setSystemUIOverlayStyle` and `SystemChrome.setEnabledSystemUIMode` is a smaller example of the same boundary — these are calls into the engine/embedder to control system UI (status bar, navigation bar) that Flutter's own render tree doesn't own.

## Key takeaways

- Flutter draws its own pixels via Skia/Impeller rather than wrapping native platform widgets — this is why UI is visually and behaviorally consistent across platforms by default.
- The embedder layer (native `Activity`/`AppDelegate`) is a thin host; it doesn't understand your widget tree at all.
- Anything that needs true native platform capability — home-screen widgets, platform-specific APIs — has to leave the framework layer entirely via a channel; see [[Platform Channels]].
