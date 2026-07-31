# Gradle

Gradle is the build system for every Android project, Flutter's included — it resolves dependencies, compiles Kotlin/Java and native code, packages resources, and produces the final APK/AAB. A Flutter project has a normal Android Gradle project living inside `android/`, configured by the Flutter tooling but otherwise a real, inspectable Gradle setup you can (and this app does) customize directly.

## In the nested app

`android/app/build.gradle.kts` is the app-module build file:

```kotlin
plugins {
    id("com.android.application")
    id("kotlin-android")
    // The Flutter Gradle Plugin must be applied after the Android and Kotlin Gradle plugins.
    id("dev.flutter.flutter-gradle-plugin")
}

android {
    namespace = "live.suture.nested"
    compileSdk = flutter.compileSdkVersion
    ndkVersion = flutter.ndkVersion

    defaultConfig {
        applicationId = "live.suture.nested"
        minSdk = flutter.minSdkVersion
        targetSdk = flutter.targetSdkVersion
        versionCode = flutter.versionCode
        versionName = flutter.versionName
    }
    ...
}

flutter {
    source = "../.."
}
```

Several SDK/version values (`flutter.compileSdkVersion`, `flutter.minSdkVersion`, etc.) are pulled from Flutter's own tooling rather than hardcoded — this is what lets `flutter upgrade` bump the effective Android SDK target without editing this file by hand. `applicationId` (`live.suture.nested`) is the package name the Play Store and Android itself use to identify this exact app, distinct from `namespace` (which scopes generated `R` class references in the code) — the two are usually the same string but serve different purposes.

**Signing** is handled by reading a git-ignored `key.properties` file rather than hardcoding credentials into the build script:

```kotlin
val keystorePropertiesFile = rootProject.file("key.properties")
val keystoreProperties = Properties()
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(FileInputStream(keystorePropertiesFile))
}

signingConfigs {
    create("release") {
        keyAlias = keystoreProperties.getProperty("keyAlias")
        keyPassword = keystoreProperties.getProperty("keyPassword")
        storeFile = keystoreProperties.getProperty("storeFile")?.let { file(it) }
        storePassword = keystoreProperties.getProperty("storePassword")
    }
}

buildTypes {
    release {
        signingConfig = signingConfigs.getByName("release")
    }
}
```

This is the standard pattern for keeping signing secrets (keystore password, key alias/password) out of version control — `key.properties` holds the actual values locally (and on CI, injected as secrets), while `build.gradle.kts` only references property *names*, safe to commit.

The top-level `android/build.gradle.kts` is much shorter — it configures repository sources and shared build directory settings for all modules, deferring almost everything else to the Flutter Gradle plugin applied per-module.

## Key takeaways

- `flutter.*` properties (`compileSdkVersion`, `minSdkVersion`, etc.) in `build.gradle.kts` come from the Flutter tool, not hardcoded numbers — this is what keeps the native Android build in sync with the Flutter SDK version in use without manual edits.
- `applicationId` is the app's permanent, store-facing identity; changing it after publishing effectively creates a new app listing — treat it as fixed once shipped.
- Never commit real signing credentials — the `key.properties`-file pattern (reading local, git-ignored property values into `signingConfigs`) is the standard way to keep a release build reproducible on any machine/CI without secrets in source control.
- Gradle plugin ordering matters — the comment `// The Flutter Gradle Plugin must be applied after the Android and Kotlin Gradle plugins` in this app's `build.gradle.kts` reflects a real constraint, not a style preference.
