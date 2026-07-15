# Kotlin — Coding Standards & State-of-the-Art Practices

> **Scope:** Kotlin 1.9+/2.x for Android and general JVM/multiplatform
> development, including Kotlin Coroutines and Jetpack Compose usage at the
> coding level.
> **Primary sources:** Kotlin official coding conventions, Android Kotlin
> style guide, `kotlinx.coroutines` documentation.
> **Relationship to general docs:** extends [general docs](../00-index.md). Where a
> general rule and a Kotlin rule conflict, the Kotlin rule wins for Kotlin.

**Rule ID prefix:** `KT`

---

## Table of contents

1. [Tooling & enforcement](#1-tooling--enforcement)
2. [Naming conventions](#2-naming-conventions)
3. [Formatting & layout](#3-formatting--layout)
4. [Language idioms & state-of-the-art features](#4-language-idioms--state-of-the-art-features)
5. [Type safety / static analysis](#5-type-safety--static-analysis)
6. [Error handling](#6-error-handling)
7. [Memory / resource management](#7-memory--resource-management)
8. [Concurrency / async](#8-concurrency--async)
9. [Security](#9-security)
10. [Testing](#10-testing)
11. [Anti-patterns](#11-anti-patterns)
12. [Quick checklist](#quick-checklist)
13. [References](#references)

---

## 1. Tooling & enforcement

**`KT-TOOL-01` (SHOULD)** Format with **ktlint** (enforces the official
Kotlin style + Android conventions) or IntelliJ's built-in formatter with a
committed `.editorconfig`; lint with **detekt** for code-smell/complexity
rules ktlint doesn't cover.

**`KT-TOOL-02` (SHOULD)** Enable **explicit API mode**
(`-Xexplicit-api=strict`) for published libraries so every public
declaration requires an explicit visibility modifier and return type —
this prevents accidental API surface leakage (`GEN-PRIN-14`).

**`KT-TOOL-03` (SHOULD)** Treat compiler warnings as errors
(`-Werror`) for new modules, adopted via baseline for legacy ones
(`GEN-TOOL-08`/`22`).

---

## 2. Naming conventions

| Construct | Convention | Example |
|-----------|------------|---------|
| Class, interface, object, enum | `UpperCamelCase` | `UserRepository`, `NetworkError` |
| Function, property, variable | `lowerCamelCase` | `fetchUser`, `isLoading` |
| Constant (`const val`, top-level/companion) | `UPPER_SNAKE_CASE` | `MAX_RETRIES` |
| Package | all-lowercase, no underscores | `com.example.network` |
| Backing property | leading underscore | `_uiState` / public `uiState` |

**`KT-NAME-01` (SHOULD)** Suffix a mutable-state-holder's public read-only
exposure with no prefix, and back it with an underscore-prefixed private
mutable property — the standard `_uiState: MutableStateFlow` /
`uiState: StateFlow` pattern (see §8).

**`KT-NAME-02` (SHOULD)** Name factory functions that look like
constructors (`UpperCamelCase`, callable as `Widget(...)`) only when they
truly behave like a constructor (return a new instance of the declared
type, no side effects) — otherwise use a `lowerCamelCase` verb name.

---

## 3. Formatting & layout

**`KT-FMT-01` (SHOULD)** 4-space indentation (official Kotlin style); no
tabs; continuation lines indented 8 spaces. Let ktlint enforce this.

**`KT-FMT-02` (SHOULD)** One statement per line; avoid semicolons; prefer
expression-body functions (`fun square(x: Int) = x * x`) over a block body
that only returns a single expression.

---

## 4. Language idioms & state-of-the-art features

**`KT-IDIOM-01` (SHOULD)** Use `data class` for value-holding types — it
generates `equals`/`hashCode`/`toString`/`copy` for free, and `copy()` is
the idiomatic way to derive a modified immutable instance.

```kotlin
data class User(val id: String, val name: String, val email: String)

val renamed = user.copy(name = "New Name")   // immutable update
```

**`KT-IDIOM-02` (SHOULD)** Use `sealed class`/`sealed interface` to model a
closed set of alternatives, and let the compiler enforce exhaustive `when`
handling (no `else` branch needed/wanted) — turns "forgot the new case"
into a compile error (`GEN-DEF-14`).

```kotlin
sealed interface UiState {
    data object Loading : UiState
    data class Loaded(val items: List<Item>) : UiState
    data class Error(val message: String) : UiState
}

fun render(state: UiState) = when (state) {   // exhaustive, no else
    UiState.Loading -> showSpinner()
    is UiState.Loaded -> showItems(state.items)
    is UiState.Error -> showError(state.message)
}
```

**`KT-IDIOM-03` (SHOULD)** Prefer `val` over `var` everywhere possible
(`GEN-PRIN-26` immutability by default); prefer immutable collection types
(`List`, `Map`, `Set`) over their mutable counterparts in signatures,
reserving `Mutable*` for genuinely-mutating internals.

**`KT-IDIOM-04` (SHOULD)** Use scope functions (`let`, `run`, `apply`,
`also`, `with`) for their specific intended purpose (null-check chaining,
object configuration, side-effecting chains), not chained arbitrarily —
overuse of scope functions to "sound idiomatic" harms readability instead of
helping it.

**`KT-IDIOM-05` (SHOULD)** Use extension functions to add focused behavior to
existing types instead of utility classes with static methods, but keep
extensions discoverable (co-located with the type they extend or in a
clearly-named file) — extensions scattered across the codebase are hard to
find at the call site.

**`KT-IDIOM-06` (SHOULD)** Use named arguments for calls with multiple
same-typed or boolean parameters, and default parameter values instead of
Java-style method overloads (`GEN-PRIN-23`).

---

## 5. Type safety / static analysis

**`KT-TYPE-01` (MUST)** Treat Kotlin's null-safety (`String` vs `String?`)
as load-bearing — don't undermine it with the not-null assertion operator
(`!!`) except where a value's non-nullness is truly guaranteed by
construction and the assertion documents that invariant (`GEN-DEF-09`).

**`KT-TYPE-02` (SHOULD)** Use safe calls (`?.`), the Elvis operator (`?:`),
and `let`/smart-casts to handle nullability at the point of use rather than
asserting past it.

```kotlin
// Bad: crashes if user is null; tells the reader nothing about why it's safe
val name = user!!.name

// Good: handles absence explicitly
val name = user?.name ?: "Unknown"
```

**`KT-TYPE-03` (SHOULD)** Use `value class` (inline classes) to give a
primitive (a `String` ID, an `Int` amount) a distinct type with zero runtime
overhead, preventing accidental mixing of semantically different values
that share a representation (`GEN-DEF-04` parse-don't-validate).

---

## 6. Error handling

Builds on [error handling](../03-error-handling.md).

**`KT-ERR-01` (SHOULD)** Use exceptions for genuinely exceptional
conditions; model expected failure as a return type — a sealed `Result`-like
type (or Kotlin's stdlib `Result<T>` for simple cases) — when the caller is
expected to handle failure as a normal outcome (`GEN-ERR-10`).

**`KT-ERR-02` (MUST NOT)** Never catch a bare `Exception`/`Throwable` and
swallow it (`catch (e: Exception) {}`) — catch the specific exception type,
or catch broadly only at a top-level boundary that logs and converts
(`GEN-ERR-03`, `GEN-ERR-11`).

**`KT-ERR-03` (SHOULD)** In coroutines, be aware that `CancellationException`
must propagate — catching `Exception` generically in a `try`/`catch` inside
a coroutine also catches cancellation, breaking structured cancellation
unless explicitly re-thrown.

```kotlin
// Bad: also swallows CancellationException, breaking coroutine cancellation
try {
    fetchData()
} catch (e: Exception) {
    logger.warn(e)
}

// Good: let cancellation propagate
try {
    fetchData()
} catch (e: CancellationException) {
    throw e
} catch (e: Exception) {
    logger.warn(e)
}
```

---

## 7. Memory / resource management

**`KT-RES-01` (MUST)** Release resources deterministically with `use { }`
(Kotlin's `Closeable`/`AutoCloseable` equivalent of `try`-with-resources) for
files, streams, and other closeable resources (`GEN-DEF-15`).

```kotlin
FileInputStream(path).use { stream ->
    // stream.close() guaranteed, even on exception
}
```

**`KT-RES-02` (MUST)** On Android, never hold a long-lived reference to an
`Activity`/`Fragment`/`View` from a singleton, static field, or a coroutine
scope that outlives the UI component — this is the classic Android memory
leak. Use `applicationContext` for anything living beyond the UI lifecycle,
and scope coroutines to the component's lifecycle (`viewModelScope`,
`lifecycleScope`) instead of a manually-managed `GlobalScope`.

---

## 8. Concurrency / async

Builds on [concurrency](../07-concurrency.md). Kotlin's primary concurrency
tool is **structured concurrency** via coroutines.

**`KT-CONC-01` (SHOULD)** Launch coroutines in a scope tied to a defined
lifetime (`viewModelScope`, `lifecycleScope`, a custom `CoroutineScope` you
cancel explicitly) — never `GlobalScope.launch`, which has no lifecycle and
leaks work that should have been cancelled.

**`KT-CONC-02` (SHOULD)** Use `Dispatchers.IO` for blocking I/O,
`Dispatchers.Default` for CPU-bound work, and `Dispatchers.Main` (or
`@MainActor`-equivalent confinement) only for UI updates — mismatching
dispatcher to workload either blocks the UI thread or wastes the IO
dispatcher's thread pool on CPU work (`GEN-CONC-16`).

**`KT-CONC-03` (MUST NOT)** Never call a blocking function directly inside a
coroutine running on `Dispatchers.Main` or without an explicit
`withContext(Dispatchers.IO)` wrapper — this is coroutine-flavored
sync-over-async and stalls the dispatcher (`GEN-CONC-13`).

**`KT-CONC-04` (SHOULD)** Model UI/reactive state with `StateFlow` (current
value + updates) and one-shot events with `SharedFlow`/`Channel` — don't
model one-shot events (navigation, snackbars) as `StateFlow`, whose
replay-latest semantics cause a re-collected event to fire again after a
process restart/config change.

**`KT-CONC-05` (SHOULD)** Use `supervisorScope`/`SupervisorJob` when
sibling coroutines' failures should not cancel each other, and a plain
`coroutineScope` (structured, all-or-nothing) when they should — pick
deliberately; the default (`coroutineScope`) cancels all siblings on any
child's failure.

---

## 9. Security

Builds on [security](../04-security.md). Kotlin/Android-specific:

**`KT-SEC-01` (MUST)** Store credentials/tokens in the **Android Keystore**
or `EncryptedSharedPreferences`, never in plain `SharedPreferences` or
hardcoded in source (`GEN-SEC-24`/`25`).

**`KT-SEC-02` (MUST)** Use `HttpsURLConnection`/OkHttp with TLS enforced and
network security config restricting cleartext traffic; pin certificates for
high-value APIs where warranted (`GEN-SEC-21`).

**`KT-SEC-03` (SHOULD)** Deserialize external JSON with a typed
`kotlinx.serialization`/Moshi/Gson model class, not a loosely-typed
`Map<String, Any>`, so malformed/unexpected fields are caught by the parser
(`GEN-DEF-04`).

**`KT-SEC-04` (MUST)** Declare only the Android permissions the app actually
uses, and request dangerous permissions at the point of use with a clear
rationale — not all at install/startup (least privilege, `GEN-SEC` §1).

---

## 10. Testing

Builds on [testing](../05-testing.md). Kotlin-specific:

**`KT-TEST-01` (SHOULD)** Use **JUnit5** or **Kotest** for unit tests, and
**MockK** (Kotlin-idiomatic, handles `final` classes/coroutines naturally)
over Mockito-for-Java for mocking.

**`KT-TEST-02` (SHOULD)** Test coroutines with `kotlinx-coroutines-test`
(`runTest`, `TestDispatcher`) so virtual time replaces real `delay()` calls
— tests stay fast and deterministic instead of sleeping real wall-clock time
(`GEN-TEST-03` Fast/Repeatable).

**`KT-TEST-03` (SHOULD)** Inject dependencies (repositories, dispatchers,
clock) via constructor parameters or a DI framework (Hilt/Koin) rather than
Kotlin `object` singletons, so tests can substitute fakes (`GEN-TEST-21`).

---

## 11. Anti-patterns

- Not-null assertion (`!!`) used routinely instead of safe calls/Elvis.
- `catch (e: Exception)` that also swallows `CancellationException`.
- `GlobalScope.launch` instead of a lifecycle-scoped `CoroutineScope`.
- Long-lived reference to an `Activity`/`Fragment`/`View` from a singleton
  or outlived coroutine scope (Android memory leak).
- Blocking calls on `Dispatchers.Main` without `withContext(Dispatchers.IO)`.
- One-shot events modeled as `StateFlow` (replays stale events).
- `var` and mutable collections used by default instead of `val`/immutable.
- Utility classes with static-style methods instead of extension functions.
- Loosely-typed `Map<String, Any>` for external/JSON data instead of a typed
  model.
- Secrets in plain `SharedPreferences` or hardcoded in source.
- Requesting all permissions at startup instead of at point of use.

---

## Quick checklist

- [ ] ktlint + detekt clean; explicit API mode for published libraries.
- [ ] `data class` for value types; `sealed class`/`sealed interface` +
      exhaustive `when` for closed alternatives.
- [ ] `val`/immutable collections preferred; `!!` avoided outside proven
      invariants.
- [ ] Coroutines scoped to a real lifecycle (`viewModelScope`/
      `lifecycleScope`), never `GlobalScope`.
- [ ] Dispatcher matches workload (`IO`/`Default`/`Main`); no blocking calls
      on `Main` without `withContext`.
- [ ] `CancellationException` re-thrown, not swallowed by broad `catch`.
- [ ] One-shot events use `SharedFlow`/`Channel`, not `StateFlow`.
- [ ] Resources closed via `use { }`; no `Activity`/`Fragment`/`View` leaked
      into a longer-lived scope.
- [ ] Secrets in Android Keystore/`EncryptedSharedPreferences`; TLS enforced;
      typed deserialization for external data.
- [ ] Permissions requested at point of use, least privilege.
- [ ] Coroutine tests use `runTest`/`TestDispatcher`; dependencies injected
      for fakeability (MockK).

---

## References

- Kotlin official coding conventions — https://kotlinlang.org/docs/coding-conventions.html
- Android, "Kotlin style guide for Android" — https://developer.android.com/kotlin/style-guide
- Kotlin docs, "Coroutines guide" — https://kotlinlang.org/docs/coroutines-guide.html
- Kotlin docs, "Null safety" — https://kotlinlang.org/docs/null-safety.html
- Kotlin docs, "Sealed classes and interfaces" — https://kotlinlang.org/docs/sealed-classes.html
- Android Developers, "Guide to app architecture" (coding-level state/lifecycle
  parts) — https://developer.android.com/topic/architecture
- ktlint — https://pinterest.github.io/ktlint/ · detekt — https://detekt.dev/
- MockK — https://mockk.io/
