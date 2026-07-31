# JNI overview

JNI (Java Native Interface) is the mechanism the JVM (and, on Android, ART) uses to call between Java/Kotlin code and native C/C++ code. It sits *underneath* both [[FFI]] and Android's NDK — when Dart's `dart:ffi` loads a `.so` library on Android, or when Kotlin code calls into a C++ library via `System.loadLibrary`, JNI (or, for Kotlin/Java → native specifically, JNI's calling conventions) is the underlying bridge making that possible.

## Not used in the nested app, directly or indirectly

Every piece of native code in this app's `android/` directory is Kotlin calling ordinary Android SDK APIs (`SQLiteDatabase`, `RemoteViews`, `AppWidgetManager`, `SharedPreferences`) — there's no C/C++ code, no NDK usage, and no `System.loadLibrary` call anywhere. Kotlin-to-Android-SDK calls don't go through JNI at all; JNI specifically concerns crossing from managed (JVM/ART) code into unmanaged native code, which this app never does.

The one place JNI-related configuration even *appears* in this project is incidental — `build.gradle.kts` sets `ndkVersion = flutter.ndkVersion`, which exists because the Flutter engine itself is partly implemented in native code (Skia/Impeller — see [[Flutter architecture]]) and needs the NDK to build/link, not because this app's own code uses JNI.

## Where it would come up

If this app (or a future feature) needed to call a genuinely native C/C++ library from Kotlin code specifically — as opposed to from Dart via `dart:ffi` — that's when hand-written JNI bindings (or a wrapper tool that generates them) would enter the picture: declaring `external fun` methods in Kotlin, implementing them in C/C++ with JNI's specific function-naming and type-marshaling conventions, and loading the compiled library with `System.loadLibrary("mylib")`. In practice, most Flutter apps that need this instead reach for `dart:ffi` directly from Dart (see [[FFI]]), skipping Kotlin/JNI entirely and calling the native library straight from the Dart side — which is the more common path specifically because it avoids needing a JNI bridge layer at all.

## Key takeaways

- JNI is the low-level mechanism under both Android's NDK/Kotlin-to-native calls and (indirectly) Dart's FFI — this app uses neither, so JNI never comes into play, even implicitly.
- The `ndkVersion` setting in this app's `build.gradle.kts` is there for the Flutter engine's own build needs, not because any of this app's own code uses native C/C++.
- For a Flutter app specifically, needing to reach a C/C++ library is usually better served by calling `dart:ffi` directly from Dart (see [[FFI]]) than by writing Kotlin-side JNI bindings — the Kotlin/JNI path is more relevant to pure-native Android development than to Flutter apps.
