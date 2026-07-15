# Swift — Coding Standards & State-of-the-Art Practices

> **Scope:** Swift 5.9+/6.x for iOS/macOS/watchOS/tvOS app and package
> development, including the Swift 6 strict-concurrency (data-race-safety)
> model.
> **Primary sources:** Swift.org API Design Guidelines, the Swift Programming
> Language book, Swift Evolution proposals (SE-xxxx), Apple Human Interface
> Guidelines (for platform conventions this doc touches only where they are
> coding-level, e.g. accessibility APIs).
> **Relationship to general docs:** extends [general docs](../00-index.md). Where a
> general rule and a Swift rule conflict, the Swift rule wins for Swift.

**Rule ID prefix:** `SWIFT`

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

**`SWIFT-TOOL-01` (SHOULD)** Format with **swift-format** (Apple's official
formatter, integrated into SwiftPM/Xcode) or **SwiftFormat**; lint with
**SwiftLint**. Commit the config so CI and every contributor enforce the same
rules (`GEN-TOOL-02`).

**`SWIFT-TOOL-02` (SHOULD)** Build with **Swift 6 language mode** (or the
Swift 5 mode with `StrictConcurrency` upcoming feature enabled) on new
projects/targets so data-race safety is checked at compile time (§8) rather
than discovered at runtime.

**`SWIFT-TOOL-03` (SHOULD)** Treat compiler warnings as errors
(`-warnings-as-errors`) for new targets, adopted via the baseline strategy for
existing ones (`GEN-TOOL-08`/`22`).

---

## 2. Naming conventions

Per the Swift API Design Guidelines — Swift's naming rules are unusually
precise and are treated as load-bearing for readability at call sites.

| Construct | Convention | Example |
|-----------|------------|---------|
| Type (`struct`/`class`/`enum`/`protocol`) | `UpperCamelCase` | `UserProfile`, `Sendable` |
| Function, method, property, variable | `lowerCamelCase` | `fetchUser`, `isEnabled` |
| Enum case | `lowerCamelCase` | `.notStarted`, `.inProgress` |
| Protocol describing a capability | `-able`/`-ing` suffix | `Equatable`, `Collection` |
| Global constant | `lowerCamelCase` (no `k`/Hungarian prefix) | `defaultTimeout` |

**`SWIFT-NAME-01` (SHOULD)** Name functions and methods so the call site
reads as a grammatical phrase; omit needless words and don't restate the
receiver's type in the method name (`APIDG`: "clarity at the point of use").

```swift
// Bad: redundant, doesn't read naturally at the call site
employees.removeEmployee(employee)

// Good: the receiver already says "employees"
employees.remove(employee)
```

**`SWIFT-NAME-02` (SHOULD)** Label closure/function parameters for clarity at
the call site, but omit a label when it would just repeat the argument's
type or an obvious noun already established by the base name.

**`SWIFT-NAME-03` (SHOULD)** Name booleans and boolean-returning
methods/properties as assertions (`isEmpty`, `hasSuffix`, `contains`), never
as a bare noun that could be misread as the value itself.

---

## 3. Formatting & layout

**`SWIFT-FMT-01` (SHOULD)** 4-space indentation (Swift community/Apple
convention); braces on the same line as the declaration (`func foo() {`).

**`SWIFT-FMT-02` (SHOULD)** Keep one property/case per line; group related
`let`/`var` declarations together; order type members consistently
(properties → initializers → public methods → private methods) and let
`// MARK: -` comments segment large types for Xcode's jump bar.

---

## 4. Language idioms & state-of-the-art features

**`SWIFT-IDIOM-01` (SHOULD)** Prefer `struct` (value type) over `class`
(reference type) unless you specifically need reference semantics
(identity, shared mutable state, inheritance) — value types are
copy-on-write, thread-safe by default when immutable, and eliminate a whole
class of aliasing bugs (`GEN-PRIN-26`).

**`SWIFT-IDIOM-02` (SHOULD)** Use `guard` for early-exit precondition checks,
not nested `if`, to keep the happy path flat (`GEN-PRIN-25`).

```swift
// Bad: nested pyramid
func process(_ order: Order?) {
    if let order {
        if order.isValid {
            // real work buried
        }
    }
}

// Good: guard keeps the happy path flat
func process(_ order: Order?) {
    guard let order, order.isValid else { return }
    // real work at top level
}
```

**`SWIFT-IDIOM-03` (SHOULD)** Use `enum` with associated values to model a
closed set of mutually exclusive states — the compiler enforces
exhaustive `switch` handling, turning "we forgot a new case" into a compile
error (`GEN-DEF-14`).

```swift
enum LoadState<T> {
    case idle
    case loading
    case loaded(T)
    case failed(Error)
}
```

**`SWIFT-IDIOM-04` (SHOULD)** Use `protocol`-oriented composition (protocols
with default implementations via extensions) over deep class hierarchies for
shared behavior (`GEN-PRIN-12` composition over inheritance).

**`SWIFT-IDIOM-05` (SHOULD)** Use optional-chaining (`?.`), `if let`/`guard
let` shorthand (Swift 5.7+: `if let value`), and nil-coalescing (`??`)
instead of force-unwrap (`!`) or force-try (`try!`) outside of tests and
truly-invariant startup code (`GEN-DEF-09`).

**`SWIFT-IDIOM-06` (CONSIDER)** Use result builders (`@resultBuilder`,
e.g. SwiftUI's `@ViewBuilder`) idiomatically when authoring DSL-like APIs,
but recognize the readability/debuggability trade-off for consumers
unfamiliar with the pattern.

---

## 5. Type safety / static analysis

**`SWIFT-TYPE-01` (SHOULD)** Let type inference work for local variables, but
annotate public API signatures explicitly — inferred types are an internal
convenience, not part of the public contract a caller should have to guess.

**`SWIFT-TYPE-02` (SHOULD)** Use `Any`/`AnyObject` only at genuine
type-erasure boundaries (e.g. bridging to Objective-C); prefer generics or
protocol existentials (`any Protocol`, Swift 5.7+) so the type system keeps
checking the code path.

**`SWIFT-TYPE-03` (SHOULD)** Mark types `Sendable` (or use `@unchecked
Sendable` with a written justification) deliberately — this is what makes
the Swift 6 concurrency checker able to prove a type is safe to share across
isolation domains (§8).

---

## 6. Error handling

Builds on [error handling](../03-error-handling.md).

**`SWIFT-ERR-01` (SHOULD)** Model recoverable failures with `throws` and a
concrete `Error`-conforming type (usually an `enum` with associated values
for context), not with `Optional` return values that conflate "no result" and
"failed" (`GEN-ERR-10`).

```swift
enum NetworkError: Error {
    case invalidURL
    case unauthorized
    case decoding(underlying: Error)
}

func fetchUser(id: String) throws -> User { ... }
```

**`SWIFT-ERR-02` (MUST NOT)** Never use `try!`/`try?` to silently discard a
recoverable error outside of tests or a deliberately-logged, justified case
(`GEN-ERR-11`). `try?` converts the error to `nil` — indistinguishable from a
legitimate absence unless the call site can't tell the difference on
purpose.

**`SWIFT-ERR-03` (SHOULD)** Prefer typed `throws` (Swift 6:
`throws(SpecificError)`) for APIs where callers benefit from knowing the
exact error type without an `as?` cast in the `catch`.

**`SWIFT-ERR-04` (SHOULD)** Never force-unwrap (`!`) a value whose absence is
a legitimately reachable runtime condition (a network response, user input,
an array index from external data) — reserve force-unwrap for invariants
the type system can't express but that are truly guaranteed by construction.

---

## 7. Memory / resource management

**`SWIFT-RES-01` (MUST)** Understand ARC (Automatic Reference Counting):
reference types are deallocated when their last strong reference drops.
Break retain cycles with `weak`/`unowned` references, most commonly in
closures capturing `self`.

```swift
// Bad: closure strongly captures self; if self holds the closure, a cycle forms
task = Task { self.reload() }

// Good: capture list breaks the cycle
task = Task { [weak self] in self?.reload() }
```

**`SWIFT-RES-02` (SHOULD)** Use `weak` when the reference may legitimately
become `nil` during the referencing object's lifetime (e.g. a delegate);
use `unowned` only when the referenced object is guaranteed to outlive the
reference — an `unowned` access after deallocation traps (crashes), so this
guarantee must be real, not assumed.

**`SWIFT-RES-03` (SHOULD)** Release resources deterministically with `defer`
for cleanup that must run on every exit path (mirrors `GEN-DEF-15`), and
conform cleanup-needing types to a pattern where `deinit` is the last-resort
safety net, not the primary cleanup mechanism for scoped resources.

---

## 8. Concurrency / async

Builds on [concurrency](../07-concurrency.md). Swift's structured-concurrency
and actor model exist specifically to eliminate data races at compile time.

**`SWIFT-CONC-01` (SHOULD)** Use `async`/`await` and structured concurrency
(`async let`, `TaskGroup`) for asynchronous work instead of completion-handler
callbacks in new code — structured concurrency makes cancellation and error
propagation automatic and composable.

**`SWIFT-CONC-02` (MUST)** Confine mutable state that is accessed from
multiple tasks to an **actor**, not a `class` guarded by ad-hoc locking — the
compiler enforces that state is only touched through the actor's isolated
context, which is exactly `GEN-CONC-02`'s "share via safe abstractions"
applied at the language level.

```swift
actor Cache {
    private var storage: [String: Data] = [:]
    func set(_ key: String, _ value: Data) { storage[key] = value }
    func get(_ key: String) -> Data? { storage[key] }
}
```

**`SWIFT-CONC-03` (MUST)** Mark UI-updating code `@MainActor` (types,
functions, or the whole `class`) rather than manually dispatching to the
main queue — the compiler then rejects, at build time, any attempt to touch
UI state from a background context.

**`SWIFT-CONC-04` (SHOULD)** Enable Swift 6's strict concurrency checking on
new modules (`SWIFT-TOOL-02`) so cross-actor data races are caught as
compile errors instead of the intermittent runtime crashes/corruption they
were in the Swift 5 opt-in model.

**`SWIFT-CONC-05` (SHOULD)** Propagate cancellation: check
`Task.isCancelled` (or let cancellable APIs throw `CancellationError`) in
long-running async work, and don't swallow `CancellationError` — let it
propagate so the caller's cancellation actually takes effect (`GEN-CONC-15`).

---

## 9. Security

Builds on [security](../04-security.md). Swift/iOS-specific:

**`SWIFT-SEC-01` (MUST)** Store credentials, tokens, and keys in the
**Keychain**, never in `UserDefaults`, plain files, or hardcoded in source
(`GEN-SEC-24`/`25`) — `UserDefaults` is an unencrypted plist on disk.

**`SWIFT-SEC-02` (MUST)** Use `URLSession` with App Transport Security
enforced (avoid `NSAllowsArbitraryLoads` exceptions); pin certificates for
high-value APIs where the threat model warrants it (`GEN-SEC-21`).

**`SWIFT-SEC-03` (SHOULD)** Validate and decode external/untrusted JSON with
`Codable` into concrete types (parse, don't validate — `GEN-DEF-04`) rather
than working with loosely-typed `[String: Any]` dictionaries throughout the
app.

**`SWIFT-SEC-04` (SHOULD)** Use `SecRandomCopyBytes` (or a library built on
it) for security-relevant randomness, never `Int.random`/`arc4random` for
tokens or keys (`GEN-SEC-22`).

---

## 10. Testing

Builds on [testing](../05-testing.md). Swift-specific:

**`SWIFT-TEST-01` (SHOULD)** Use **Swift Testing** (the modern
`@Test`/`#expect` framework) or **XCTest** for unit/integration tests;
prefer dependency injection (protocol-typed dependencies) over singletons so
network/persistence layers can be faked in tests (`GEN-TEST-21`).

**`SWIFT-TEST-02` (SHOULD)** Test actors and async code with `await` in the
test body directly; avoid sleeping/polling for async completion — use the
async APIs (`Task`, `withCheckedContinuation` fakes, or test-scheduler
utilities) to make tests deterministic (`GEN-TEST-03` Repeatable).

**`SWIFT-TEST-03` (CONSIDER)** Use snapshot testing (`swift-snapshot-testing`)
for view-layer regressions, reviewed deliberately rather than blindly
re-recorded on every failure (`GEN-TEST-12` §Other test types).

---

## 11. Anti-patterns

- Force-unwrap (`!`) or `try!`/`try?` on a value/error that can legitimately
  occur at runtime.
- Retain cycles from closures capturing `self` strongly with no `[weak self]`.
- Ad-hoc locks/`DispatchQueue` synchronization around shared mutable state
  instead of an `actor`.
- Manual `DispatchQueue.main.async` for UI updates instead of `@MainActor`.
- Secrets/tokens stored in `UserDefaults` or hardcoded in source.
- Loosely-typed `[String: Any]` used throughout instead of `Codable` models.
- Deep class inheritance where protocol composition would do.
- Swallowed `CancellationError`, breaking cooperative cancellation.
- Class used where a value-semantic `struct` would remove aliasing risk.

---

## Quick checklist

- [ ] swift-format/SwiftFormat + SwiftLint in CI; Swift 6 / strict
      concurrency mode on new targets.
- [ ] API-Design-Guidelines naming; call sites read as phrases; booleans read
      as assertions.
- [ ] `struct` preferred over `class` unless reference semantics are needed.
- [ ] `guard` for early exits; exhaustive `switch` over enums with associated
      values.
- [ ] No force-unwrap/`try!`/`try?` on runtime-reachable failure; typed
      `throws` where it helps callers.
- [ ] ARC retain cycles broken with `[weak self]`/`unowned`; `defer` for
      deterministic cleanup.
- [ ] Shared mutable state isolated in an `actor`; UI code `@MainActor`; no
      manual dispatch-queue synchronization.
- [ ] Cancellation propagated (`Task.isCancelled`/`CancellationError` not
      swallowed).
- [ ] Secrets in Keychain, never `UserDefaults`/hardcoded; ATS enforced;
      `Codable` for external data; CSPRNG for tokens.
- [ ] Dependencies injected as protocols for fakeability; async tests await
      directly (no sleep/poll); snapshot tests reviewed, not blind-approved.

---

## References

- Swift.org, "API Design Guidelines" — https://www.swift.org/documentation/api-design-guidelines/
- *The Swift Programming Language* — https://docs.swift.org/swift-book/
- Swift Evolution — https://github.com/swiftlang/swift-evolution
- Swift.org, "Concurrency" (structured concurrency, actors, Sendable) — https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/
- Migrating to Swift 6 — https://www.swift.org/migration/documentation/migrationguide/
- SwiftLint — https://github.com/realm/SwiftLint
- swift-format — https://github.com/swiftlang/swift-format
- Apple, "Keychain Services" — https://developer.apple.com/documentation/security/keychain_services
- Apple, "Swift Testing" — https://developer.apple.com/documentation/testing
