# Dart & Flutter — Coding Standards & State-of-the-Art Practices

> **Scope:** Dart 3.x (sound null safety, records, patterns, sealed classes)
> and Flutter for cross-platform mobile/desktop/web UI development.
> **Primary sources:** Effective Dart, Dart language specification, Flutter
> official documentation and API guidelines.
> **Relationship to general docs:** extends [general docs](../00-index.md). Where a
> general rule and a Dart/Flutter rule conflict, the Dart/Flutter rule wins for
> Dart/Flutter code.

**Rule ID prefix:** `DART`

---

## Table of contents

1. [Tooling & enforcement](#1-tooling--enforcement)
2. [Naming conventions](#2-naming-conventions)
3. [Formatting & layout](#3-formatting--layout)
4. [Language idioms & state-of-the-art features](#4-language-idioms--state-of-the-art-features)
5. [Type safety / static analysis](#5-type-safety--static-analysis)
6. [Error handling](#6-error-handling)
7. [Widget & state management](#7-widget--state-management)
8. [Concurrency / async](#8-concurrency--async)
9. [Security](#9-security)
10. [Testing](#10-testing)
11. [Anti-patterns](#11-anti-patterns)
12. [Quick checklist](#quick-checklist)
13. [References](#references)

---

## 1. Tooling & enforcement

**`DART-TOOL-01` (MUST)** Format with `dart format` (the single canonical
formatter — there is no configuration to argue about) and lint with `dart
analyze` using `package:lints`/`package:flutter_lints` as the base rule set
(`GEN-TOOL-02`).

**`DART-TOOL-02` (SHOULD)** Enable sound null safety (default since Dart 2.12
and mandatory for current SDKs) and keep `analysis_options.yaml` committed
so every contributor and CI share the same lint configuration.

**`DART-TOOL-03` (SHOULD)** Run `dart fix --apply` for lint-suggested
mechanical fixes as part of the pre-merge routine, but review the diff — not
every suggested fix is behavior-neutral.

---

## 2. Naming conventions

Per Effective Dart:

| Construct | Convention | Example |
|-----------|------------|---------|
| Class, enum, extension, typedef | `UpperCamelCase` | `UserRepository`, `ConnectionState` |
| Library, package, file, directory | `lowercase_with_underscores` | `user_repository.dart` |
| Variable, function, method, parameter | `lowerCamelCase` | `fetchUser`, `isLoading` |
| Constant | `lowerCamelCase` (not `SCREAMING_CAPS` — Dart convention differs from many languages) | `defaultTimeout` |
| Import prefix | `lowercase_with_underscores` | `import 'dart:math' as math;` |

**`DART-NAME-01` (SHOULD)** Name booleans and boolean-returning members as
assertions (`isEmpty`, `hasError`, `canSubmit`) per Effective Dart's
"Do name a boolean property/variable using a verb phrase" guidance.

**`DART-NAME-02` (SHOULD)** Prefix a private-to-library member with a single
leading underscore (`_controller`); Dart privacy is library-scoped, not
class-scoped, so treat one file/library as the encapsulation boundary.

---

## 3. Formatting & layout

**`DART-FMT-01` (MUST)** 2-space indentation, trailing commas in multi-line
argument/parameter lists (this is what lets `dart format` produce a clean
one-item-per-line layout for widget trees) — always run `dart format`
rather than hand-formatting.

**`DART-FMT-02` (SHOULD)** Keep widget-building methods/functions small and
extract sub-trees into their own `Widget` (function or class) once a
`build()` method's nesting gets deep — a deeply nested `build()` is exactly
the "flat happy path" violation from `GEN-PRIN-25` applied to widget trees.

---

## 4. Language idioms & state-of-the-art features

**`DART-IDIOM-01` (SHOULD)** Use Dart 3 **records** (`(String, int)` or
named `({String name, int age})`) for lightweight, multi-value returns
instead of a single-use class or an out-parameter, and use **patterns**
(`switch` with destructuring, `if-case`) to consume them.

```dart
(double, double) minMax(List<double> values) {
  return (values.reduce(min), values.reduce(max));
}

final (lo, hi) = minMax(samples);
```

**`DART-IDIOM-02` (SHOULD)** Use `sealed class` (Dart 3) to model a closed
set of alternatives and exhaustive `switch` expressions over them — the
analyzer flags a `switch` that doesn't cover every subtype, turning "we
forgot the new case" into a compile-time error (`GEN-DEF-14`).

```dart
sealed class UiState {}
class Loading extends UiState {}
class Loaded extends UiState { final List<Item> items; Loaded(this.items); }
class Failed extends UiState { final String message; Failed(this.message); }

String describe(UiState state) => switch (state) {
  Loading() => 'Loading…',
  Loaded(:final items) => '${items.length} items',
  Failed(:final message) => 'Error: $message',
};
```

**`DART-IDIOM-03` (SHOULD)** Prefer immutable `final` fields and `const`
constructors for widgets and value objects wherever the data doesn't change
— `const` widgets let Flutter skip rebuilding them entirely, which is
simultaneously a correctness idiom (`GEN-PRIN-26`) and a performance one
(§7).

**`DART-IDIOM-04` (SHOULD)** Use cascade notation (`..`) for a sequence of
operations on the same receiver, and the null-aware cascade (`?..`)
when the receiver may be `null` — but don't chain so many operations that
readability suffers; a long cascade is a smell the same way a long fluent
chain is (`GEN-PRIN-13` Law of Demeter caveat).

**`DART-IDIOM-05` (SHOULD)** Use extension methods to add focused,
discoverable behavior to existing types (including third-party/SDK types)
instead of wrapper/utility classes.

---

## 5. Type safety / static analysis

**`DART-TYPE-01` (MUST)** Treat sound null safety as load-bearing: don't
use the null-assertion operator (`!`) to silence the analyzer on a value
that can legitimately be `null` at runtime — handle it with `?.`, `??`, or
an explicit `if (value != null)` check (`GEN-DEF-09`).

**`DART-TYPE-02` (SHOULD)** Avoid `dynamic` except at genuine
interop/reflection boundaries (e.g. decoding raw JSON before mapping to a
typed model); prefer `Object?` plus explicit casts/type checks so the
analyzer keeps checking the code path (mirrors `SWIFT-TYPE-02`/`KT` `Any`
guidance).

**`DART-TYPE-03` (SHOULD)** Model API responses and other external data with
typed classes (hand-written, `json_serializable`, or `freezed`), not raw
`Map<String, dynamic>` passed around the app — parse once at the boundary
(`GEN-DEF-04`).

---

## 6. Error handling

Builds on [error handling](../03-error-handling.md).

**`DART-ERR-01` (SHOULD)** Throw for genuinely exceptional conditions;
model expected failure as a return type (a `Result`-like sealed class, or a
nullable/`Either`-style value) when the caller is expected to handle failure
as a normal branch, not an exception path (`GEN-ERR-10`).

**`DART-ERR-02` (MUST NOT)** Never use a bare `catch (e) {}` that swallows
the error silently; catch specific exception types, and if you must catch
broadly, log and either rethrow or convert to a domain error at a defined
boundary (`GEN-ERR-11`, `GEN-ERR-13`).

**`DART-ERR-03` (SHOULD)** Set `FlutterError.onError` and
`PlatformDispatcher.instance.onError` (or an equivalent crash-reporting SDK
hook) at app startup so uncaught widget-build and async errors are captured
centrally instead of only appearing as a red error screen in debug builds
(`GEN-ERR-12` top-level catch-all).

**`DART-ERR-04` (SHOULD)** Never let a `Future` reject unobserved — every
`Future`/`async` call site should either `await` it, attach `.catchError`,
or be explicitly fire-and-forget with a documented reason; an unhandled
`Future` rejection is a silently-dropped error (`GEN-ERR-11`).

---

## 7. Widget & state management

**`DART-STATE-01` (SHOULD)** Keep widgets as close to stateless as possible;
lift state up to the smallest common ancestor that actually needs it, and
pass data down via constructor parameters — this is Separation of Concerns
(`GEN-PRIN-05`) applied to the widget tree.

**`DART-STATE-02` (SHOULD)** Pick one state-management approach deliberately
(Provider, Riverpod, Bloc, or `ValueNotifier`/`ChangeNotifier` for small
scopes) and apply it consistently across the app — mixing several
approaches in one codebase multiplies the mental model a reader needs.

**`DART-STATE-03` (SHOULD)** Dispose everything that needs disposing
(`AnimationController`, `TextEditingController`, `StreamSubscription`,
`FocusNode`) in the owning `State`'s `dispose()` — an undisposed controller
is a resource/memory leak with the same failure mode as an unclosed file
handle (`GEN-DEF-15`).

```dart
class _MyFormState extends State<MyForm> {
  final _controller = TextEditingController();

  @override
  void dispose() {
    _controller.dispose();   // released on every path, incl. navigation away
    super.dispose();
  }
}
```

**`DART-STATE-04` (SHOULD)** Use `const` constructors for widget subtrees
that never change, and split large `build()` methods into smaller `const`-
constructible widgets — this lets Flutter's widget-diffing skip whole
subtrees on rebuild instead of re-evaluating them (`GEN-PERF-08` reduce
allocation in hot paths, applied to the widget-rebuild loop).

---

## 8. Concurrency / async

Builds on [concurrency](../07-concurrency.md). Dart is single-threaded per
isolate with an event loop; true parallelism requires separate isolates.

**`DART-CONC-01` (SHOULD)** Use `async`/`await` and `Future`/`Stream` for
I/O-bound concurrency on the main isolate — this is Dart's equivalent of an
event loop and follows the same "never block it" rule as any other async
runtime (`GEN-CONC-14`).

**`DART-CONC-02` (MUST NOT)** Never run CPU-heavy synchronous work
(large JSON parsing, image processing, heavy computation) directly on the
UI isolate's event loop — it blocks frame rendering. Offload to a separate
**isolate** via `compute()`/`Isolate.run()`.

**`DART-CONC-03` (SHOULD)** Cancel `StreamSubscription`s and `Timer`s in
`dispose()` (§7) and check `mounted` before calling `setState` after an
`await` — the widget may have been unmounted while the async operation was
in flight, and calling `setState` on an unmounted widget throws.

```dart
Future<void> _load() async {
  final data = await repository.fetch();
  if (!mounted) return;         // widget may be gone by the time await returns
  setState(() => _data = data);
}
```

**`DART-CONC-04` (SHOULD)** Use `StreamController.broadcast()` only when
multiple listeners are genuinely expected; a single-subscription stream
that's accidentally listened to twice throws — pick the stream type
deliberately.

---

## 9. Security

Builds on [security](../04-security.md). Flutter-specific:

**`DART-SEC-01` (MUST)** Store credentials/tokens in platform secure storage
(`flutter_secure_storage`, backed by Keychain/Keystore), never in
`SharedPreferences`/plain files or hardcoded in source (`GEN-SEC-24`/`25`).

**`DART-SEC-02` (MUST)** Use `Dio`/`http` with TLS enforced and certificate
pinning for high-value APIs; don't disable certificate validation
(`badCertificateCallback`) outside of a scoped test/debug build
(`GEN-SEC-21`).

**`DART-SEC-03` (SHOULD)** Decode external JSON via typed models
(`json_serializable`/`freezed`), not raw `Map<String, dynamic>` passed
around business logic (`GEN-DEF-04`, `DART-TYPE-03`).

---

## 10. Testing

Builds on [testing](../05-testing.md). Flutter-specific:

**`DART-TEST-01` (SHOULD)** Use `flutter_test`'s widget tests
(`testWidgets`, `WidgetTester`) for component behavior, `package:test` for
pure-Dart unit tests, and `integration_test` sparingly for critical
end-to-end flows (`GEN-TEST-02` pyramid/trophy shape).

**`DART-TEST-02` (SHOULD)** Use `pumpAndSettle()`/`pump(duration)`
deliberately rather than real `Future.delayed` sleeps in tests, and inject
fakes for repositories/clocks so widget tests stay deterministic
(`GEN-TEST-03` Fast/Repeatable).

**`DART-TEST-03` (CONSIDER)** Use golden-file (snapshot) tests for
pixel-level widget regressions, reviewed deliberately on diff, not
blindly re-recorded (`GEN-TEST-12`).

---

## 11. Anti-patterns

- Null-assertion (`!`) used routinely instead of `?.`/`??`/explicit checks.
- Raw `Map<String, dynamic>` passed through business logic instead of typed
  models.
- Deeply nested `build()` methods instead of extracted sub-widgets.
- Mixing multiple state-management approaches in the same codebase.
- Undisposed `AnimationController`/`TextEditingController`/
  `StreamSubscription`/`FocusNode`.
- CPU-heavy synchronous work run on the UI isolate instead of `compute()`/
  a separate isolate.
- `setState` called after `await` without checking `mounted`.
- Bare `catch (e) {}` swallowing errors silently.
- Unobserved rejected `Future`s.
- Secrets in `SharedPreferences` or hardcoded in source.
- Certificate validation disabled outside a scoped test/debug build.

---

## Quick checklist

- [ ] `dart format` + `dart analyze` (flutter_lints) clean.
- [ ] Sound null safety respected; `!` avoided outside proven invariants.
- [ ] Records/patterns/`sealed class` used for structured data and closed
      alternatives with exhaustive `switch`.
- [ ] External data modeled with typed classes, not raw
      `Map<String, dynamic>`.
- [ ] Widgets small, mostly stateless/`const`; state lifted to the smallest
      common owner.
- [ ] One state-management approach used consistently.
- [ ] All controllers/subscriptions/timers disposed in `dispose()`;
      `mounted` checked after `await` before `setState`.
- [ ] No CPU-heavy work on the UI isolate; `compute()`/`Isolate.run()` used.
- [ ] No swallowed errors; `FlutterError.onError`/`PlatformDispatcher.onError`
      wired at startup; no unobserved rejected `Future`s.
- [ ] Secrets in secure storage; TLS/cert pinning enforced.
- [ ] Widget tests deterministic (fakes, `pumpAndSettle`); goldens reviewed
      on diff.

---

## References

- Dart, "Effective Dart" — https://dart.dev/effective-dart
- Dart language tour — https://dart.dev/language
- Dart docs, "Sound null safety" — https://dart.dev/null-safety
- Dart docs, "Pattern matching" — https://dart.dev/language/patterns
- Flutter docs, "State management" — https://docs.flutter.dev/data-and-backend/state-mgmt/intro
- Flutter docs, "Concurrency in Flutter" (isolates) — https://docs.flutter.dev/perf/isolates
- Flutter docs, "Error handling in Flutter" — https://docs.flutter.dev/testing/errors
- Flutter docs, "Widget testing" — https://docs.flutter.dev/testing/overview
- flutter_secure_storage — https://pub.dev/packages/flutter_secure_storage
