# Kotlin basics

The nested app's `android/` directory is real, idiomatic Kotlin — not boilerplate — and its native widget implementation is a good tour of the language features that show up constantly in everyday Android code.

## Data classes

`data class` auto-generates `equals`/`hashCode`/`toString`/`copy` from constructor properties — used throughout for small value-holding types:

```kotlin
// NestedRemoteViewsService.kt
data class WidgetTask(val id: Long, val name: String, val priority: Int, val isCompleted: Boolean)

// WidgetPageResolver.kt
data class HeaderState(val title: String, val currentFolderId: Long, val showArrows: Boolean, ...)
```

Compare to Dart's `TreeNode` (`lib/models/tree_node.dart`), which hand-writes its own `copyWith`/equality — Kotlin's `data class` gives you that for free.

## `object` — singletons without ceremony

`object Foo { ... }` declares a class with exactly one instance, created lazily on first access — no `getInstance()` boilerplate needed:

```kotlin
// WidgetPrefs.kt
object WidgetPrefs {
    private const val PREFS_NAME = "nested_widget_prefs"
    fun getThemeKey(context: Context): String = prefs(context).getString(KEY_THEME, "light") ?: "light"
}
```

`WidgetTheme`, `WidgetPageResolver`, and `NodeDbContract` are all `object`s in this codebase — each is a natural singleton (shared config/logic, no per-instance state), which is exactly what `object` is for.

## `companion object` — Java-style statics, scoped to a class

Where a Kotlin file needs class-scoped constants/functions rather than a fully separate singleton, `companion object` does it:

```kotlin
// NestedWidgetProvider.kt
class NestedWidgetProvider : AppWidgetProvider() {
    companion object {
        const val EXTRA_APPWIDGET_ID = AppWidgetManager.EXTRA_APPWIDGET_ID
        fun refreshAllWidgets(context: Context) { ... }
    }
}
```

This is why `NestedWidgetProvider.refreshAllWidgets(context)` can be called from `MainActivity.kt`, `WidgetRowActionReceiver.kt`, and `AddTaskActivity.kt` without ever instantiating a `NestedWidgetProvider` — it's called on the class itself, the same way a Dart `static` method would be.

## `when` — pattern-matching expressions, not just switch statements

`when` returns a value (unlike Java's `switch` statement) and is used as an expression constantly:

```kotlin
// WidgetTheme.kt
fun resolve(context: Context): WidgetThemeColors = when (WidgetPrefs.getThemeKey(context)) {
    "dark" -> WidgetThemeColors(backgroundDrawableRes = R.drawable.widget_bg_dark, ...)
    "amoled" -> WidgetThemeColors(backgroundDrawableRes = R.drawable.widget_bg_amoled, ...)
    else -> WidgetThemeColors(backgroundDrawableRes = R.drawable.widget_bg_light, ...)
}
```

## Null safety — `?`, `?:`, `?.`, and the elvis operator

Kotlin's null safety is as central as Dart's — this codebase leans on it constantly:

```kotlin
// WidgetPrefs.kt — elvis operator (?:) supplies a fallback when the left side is null
fun getThemeKey(context: Context): String = prefs(context).getString(KEY_THEME, "light") ?: "light"

// MainActivity.kt — safe call (?.) plus let, only runs the block if non-null
extractFolderId(intent)?.let { pendingFolderId = it }
```

## Scope functions — `apply`, `let`, `use`

Kotlin's scope functions (`apply`, `let`, `run`, `with`, `also`) each configure/transform a receiver slightly differently; two show up repeatedly here:

`apply` runs a block against a receiver and returns the receiver itself — perfect for object configuration:

```kotlin
// NestedWidgetProvider.kt
val addIntent = Intent(context, AddTaskActivity::class.java).apply {
    putExtra(EXTRA_APPWIDGET_ID, appWidgetId)
    putExtra(EXTRA_PARENT_ID, state.currentFolderId)
    addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
}
```

`use` (Kotlin's `try`-with-resources equivalent) guarantees a `Closeable` — here, a `SQLiteDatabase`/`Cursor` — gets closed even if an exception is thrown inside the block:

```kotlin
// NodeDbHelper.kt
val db = openDb(readOnly = true)
db.use {
    val cursor = it.query(TABLE_NODES, ...)
    cursor.use { c -> while (c.moveToNext()) { ... } }
}
```

## Extension-style member access and string templates

String templates (`"$key"`, `"${expr}"`) replace manual concatenation everywhere native strings are built, e.g. `WidgetPrefs.kt`'s `private fun key(appWidgetId: Int) = "current_folder_$appWidgetId"`.

## Key takeaways

- `data class` for value-holding types, `object` for true singletons, `companion object` for class-scoped statics — this codebase uses all three deliberately and distinctly, not interchangeably.
- `when` as an expression (returning a value, exhaustive over sealed/enum-like inputs) is idiomatic Kotlin — prefer it over a chain of `if`/`else if` whenever you're mapping one value to another, as `WidgetTheme.resolve` does.
- `?:` (elvis) for fallback values, `?.let { }` for "only do this if non-null" — both appear constantly in this codebase's null-handling and are worth having as reflexes.
- `.use { }` is the idiomatic way to guarantee a `Cursor`/`SQLiteDatabase`/any `Closeable` is closed — `NodeDbHelper.kt`'s consistent use of it is what keeps its manual SQLite access leak-free without try/finally boilerplate.
