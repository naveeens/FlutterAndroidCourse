# FFI

Dart's `dart:ffi` lets Dart code call into native C-compatible shared libraries (`.so` on Android, `.dylib`/`.framework` on iOS/macOS) directly, in-process, without going through a platform channel's async message-passing — useful when you need raw performance (heavy numeric computation, an existing C/C++/Rust library you want to reuse) or synchronous native calls that a platform channel's inherently async model can't offer.

## Not used in the nested app — and why platform channels were already sufficient

Every native capability this app needs — refreshing/theming the home-screen widget, reading a cold-start deep-link folder id (see [[MethodChannel]]) — is small, infrequent, and naturally asynchronous (a widget refresh doesn't need to block the UI thread waiting for a synchronous return value). That's exactly the profile [[Platform Channels]] are built for, and there's no C/C++ library or performance-critical numeric workload anywhere in this codebase that would justify dropping to FFI instead.

The clearest way to see the distinction: this app's actual Kotlin code (`NodeDbHelper.kt`, `WidgetTheme.kt`, `NestedWidgetProvider.kt`) is ordinary Android SDK code — calls into `SQLiteDatabase`, `RemoteViews`, `AppWidgetManager`, all of which are Kotlin/Java APIs a platform channel reaches naturally. None of it is a C library that would need FFI's specific capability (calling into compiled native code with a C ABI) rather than a channel's capability (calling into Kotlin/Java code via message passing).

## What would actually require FFI here

If this app added, say, a custom text-diffing or compression routine for GitHub sync reconciliation and wanted to reuse an existing high-performance C library rather than a pure-Dart implementation, FFI would be the tool:

```dart
import 'dart:ffi';

typedef DiffNative = Int32 Function(Pointer<Utf8> a, Pointer<Utf8> b);
typedef DiffDart = int Function(Pointer<Utf8> a, Pointer<Utf8> b);

final dylib = DynamicLibrary.open('libdiff.so');
final diff = dylib.lookupFunction<DiffNative, DiffDart>('diff');
```

That's a meaningfully different integration shape than anything this app has — no `MethodChannel`, no async round-trip, no Kotlin code at all involved; Dart calls directly into a compiled shared library's exported C function, synchronously, in the same process.

## Key takeaways

- FFI bypasses platform channels entirely for calling native C-ABI code directly and synchronously — it solves a different problem than this app has: performance-critical or existing-C-library integration, not "call into the Android SDK."
- Reaching for platform channels (as this app does) is correct whenever the native code you need is itself Kotlin/Java/Swift/Objective-C using platform SDK APIs — that's precisely what channels are designed to reach.
- FFI becomes relevant only when you specifically need a compiled C-ABI library and want to avoid channel message-passing overhead/async — a genuinely different use case from anything in this codebase today.
