# C# / .NET — Coding Standards & State-of-the-Art Practices

> **Scope:** Modern C# on current .NET (LTS .NET 8 and later; .NET 6 also
> supported). Language-version examples target C# 12–13. Note each C# version is
> tied to a runtime — e.g. **C# 12 ships with .NET 8, C# 13 with .NET 9** — so
> features below require the matching SDK/target framework.
> **Primary sources:** Microsoft .NET Coding Conventions, Framework Design
> Guidelines, .NET Runtime/Roslyn style.
> **Relationship to general docs:** extends [general docs](../00-index.md). Where a
> general rule and a C# rule conflict, the C# rule wins for C#.

**Rule ID prefix:** `CS`

---

## Table of contents

1. [Tooling & enforcement](#1-tooling--enforcement)
2. [Naming conventions](#2-naming-conventions)
3. [Formatting & layout](#3-formatting--layout)
4. [Types & language keywords](#4-types--language-keywords)
5. [`var` and implicit typing](#5-var-and-implicit-typing)
6. [Nullable reference types](#6-nullable-reference-types)
7. [Modern language features](#7-modern-language-features)
8. [Immutability: records, readonly, init](#8-immutability-records-readonly-init)
9. [Strings](#9-strings)
10. [Collections & LINQ](#10-collections--linq)
11. [Exceptions & error handling](#11-exceptions--error-handling)
12. [Resource management: IDisposable & using](#12-resource-management-idisposable--using)
13. [Async/await](#13-asyncawait)
13a. [Threading & synchronization](#13a-threading--synchronization)
13b. [Dependency injection](#13b-dependency-injection)
14. [Delegates, events, lambdas](#14-delegates-events-lambdas)
15. [Namespaces & usings](#15-namespaces--usings)
16. [API & library design](#16-api--library-design)
17. [Security](#17-security)
18. [Testing](#18-testing)
19. [Anti-patterns](#19-anti-patterns)
20. [Quick checklist](#quick-checklist)
21. [References](#references)

---

## 1. Tooling & enforcement

**`CS-TOOL-01` (MUST)** Use an `.editorconfig` to define and enforce style; ship
the `dotnet/docs` config or a team variant. Visual Studio, Rider, and `dotnet
format` apply it automatically. This makes style objective and CI-enforceable.

**`CS-TOOL-02` (SHOULD)** Enable .NET analyzers and `TreatWarningsAsErrors`. Turn
on `<AnalysisMode>Recommended</AnalysisMode>` (or `All` for strict projects), or
the equivalent `<AnalysisLevel>latest-recommended</AnalysisLevel>` /
`latest-all`. Address analyzer diagnostics rather than suppressing them.

**`CS-TOOL-03` (SHOULD)** Enable nullable reference types project-wide
(`<Nullable>enable</Nullable>`) — see §6.

**`CS-TOOL-04` (SHOULD)** Use `dotnet format` (and/or StyleCop/Roslynator) in
pre-commit hooks and CI so reviews focus on design, not formatting.

---

## 2. Naming conventions

Per Microsoft Framework Design Guidelines:

| Construct | Convention | Example |
|-----------|------------|---------|
| Namespace, class, struct, record, enum, method, property, event | **PascalCase** | `OrderService`, `TotalPrice` |
| Interface | **PascalCase** with `I` prefix | `IRepository` |
| Public/protected fields, constants | PascalCase | `MaxRetries` |
| Private fields | `_camelCase` (leading underscore) | `_retryCount` |
| Local variables, parameters | **camelCase** | `itemCount`, `userName` |
| Type parameters (generics) | PascalCase, `T` prefix | `T`, `TKey`, `TResult` |
| Async methods | PascalCase + `Async` suffix | `LoadAsync` |

**`CS-NAME-01` (SHOULD)** Use PascalCase for record **primary-constructor
parameters** that become public properties; camelCase for class/struct
primary-constructor parameters used as fields. (Per MS conventions.)

**`CS-NAME-02` (SHOULD)** Don't abbreviate; use meaningful names. Acronyms: 2
letters all-caps (`IO`), 3+ Pascal-cased (`Html`, `Xml`). Don't use Hungarian
notation.

**`CS-NAME-03` (SHOULD)** Boolean members read as assertions (`IsEnabled`,
`HasItems`, `CanExecute`).

---

## 3. Formatting & layout

**`CS-FMT-01` (SHOULD)** Use **Allman braces** (opening brace on its own line) and
4-space indentation (no tabs). One statement and one declaration per line.

**`CS-FMT-02` (SHOULD)** Always brace control blocks, even single statements
(prevents the "goto fail" edit hazard; improves diff clarity).

**`CS-FMT-03` (SHOULD)** Add a blank line between method/property definitions;
break long statements across lines; place line breaks **before** binary operators
when wrapping. Use parentheses to make precedence explicit.

```csharp
public sealed class OrderService
{
    private readonly IRepository _repository;

    public OrderService(IRepository repository)
    {
        _repository = repository;
    }

    public async Task<Order> GetAsync(int id)
    {
        if (id <= 0)
        {
            throw new ArgumentOutOfRangeException(nameof(id));
        }

        return await _repository.FindAsync(id);
    }
}
```

---

## 4. Types & language keywords

**`CS-TYPE-01` (SHOULD)** Use the **language keyword** for built-in types, not the
CLR/runtime type name: `string` not `String`, `int` not `Int32`, `bool` not
`Boolean`. (Explicit MS guidance; includes `nint`/`nuint`.)

**`CS-TYPE-02` (SHOULD)** Prefer `int` over unsigned types for general counting —
`int` interoperates better with the BCL and most APIs; reserve unsigned types for
bit manipulation/interop where the domain demands it. (MS guidance.)

**`CS-TYPE-03` (SHOULD)** Choose `struct` (value type) only for small, immutable,
short-lived values with value semantics; otherwise use `class`. Large or mutable
structs cause subtle copy bugs and copying overhead.

**`CS-TYPE-04` (SHOULD)** Seal classes by default (`sealed`) unless designed for
inheritance — it documents intent, can improve performance, and avoids fragile
base-class problems (`GEN-PRIN-12`).

---

## 5. `var` and implicit typing

**`CS-VAR-01` (SHOULD)** Use `var` **only when the type is obvious** from the
right-hand side (a `new`, a cast, or a literal). Don't use `var` when the type
isn't apparent. (MS guidance.)

```csharp
var message = "hello";                       // obvious -> var OK
var customer = new Customer();               // obvious -> var OK
int count = ParseCount(input);               // not obvious -> name the type
```

**`CS-VAR-02` (SHOULD)** Use `var` for `foreach`... **no** — use an explicit type
for the `foreach` loop variable when the element type isn't obvious from the
collection (MS guidance). Use `var` for LINQ results that are anonymous/nested
generic types (where it's required or far more readable).

**Pros of `var`:** less noise, easier to read when the type is obvious, required
for anonymous types.
**Cons of `var`:** hides the type when it isn't obvious, hurting readability and
review (especially outside an IDE) — hence the "only when obvious" rule.

---

## 6. Nullable reference types

**`CS-NULL-01` (SHOULD)** Enable nullable reference types (`<Nullable>enable
</Nullable>`) and treat nullable warnings as errors.

**Why:** NRTs move null-handling from runtime `NullReferenceException` to
compile-time checks. The compiler tracks which references may be null and forces
you to handle them — eliminating the most common .NET crash (`GEN-DEF-11`).

**`CS-NULL-02` (SHOULD)** Express intent in the type: `Customer?` means "may be
null"; `Customer` means "never null." Don't suppress with the null-forgiving
operator `!` except when you genuinely know better than the compiler — and comment
why.

**`CS-NULL-03` (SHOULD)** Validate public-API arguments with guard clauses using the
modern static throw-helpers: **`ArgumentNullException.ThrowIfNull(arg)`** (.NET 6+)
for null checks, and **`ArgumentException.ThrowIfNullOrEmpty(arg)`** /
`ThrowIfNullOrWhiteSpace` (.NET 7+) for string checks. (Note `ThrowIfNullOrEmpty`
is a member of `ArgumentException`, not `ArgumentNullException`.)

```csharp
public void Process(Order order)
{
    ArgumentNullException.ThrowIfNull(order);   // fail fast at the boundary
    // ... order is non-null below
}
```

---

## 7. Modern language features

**`CS-LANG-01` (SHOULD)** Prefer modern, expressive constructs that reduce
boilerplate and bugs:

- **Pattern matching & switch expressions** — concise, exhaustive branching.
- **Target-typed `new()`** when the variable type is explicit: `Customer c = new();`
- **Collection expressions** (`int[] xs = [1, 2, 3];`).
- **`nameof`** instead of string literals for member names (refactor-safe).
- **Expression-bodied members** for trivial one-liners.
- **Tuples** for lightweight multiple returns (name the elements).
- **`required` members** and `init` setters for immutable construction.

```csharp
// Exhaustive switch expression — compiler warns on unhandled cases
public static decimal Discount(CustomerTier tier) => tier switch
{
    CustomerTier.Standard => 0.0m,
    CustomerTier.Gold     => 0.1m,
    CustomerTier.Platinum => 0.2m,
    _ => throw new ArgumentOutOfRangeException(nameof(tier)),
};
```

**`CS-LANG-02` (SHOULD)** Make `switch` expressions over enums exhaustive
(`GEN-DEF-14`) so adding an enum value surfaces every place that must change.

---

## 8. Immutability: records, readonly, init

**`CS-IMM-01` (SHOULD)** Use **`record`** (or `record struct`) for immutable data
types and DTOs/value objects — you get value equality, `with`-expressions, and
concise declaration for free.

> **Caveat — identity vs value:** records use **value-based equality**, which is
> wrong for domain **entities** that have a distinct identity (two customers with
> the same fields are not the same customer). Use a `class` with identity-based
> equality for identity-rich entities; reserve records for value objects where two
> equal-valued instances are genuinely interchangeable.

**`CS-IMM-02` (SHOULD)** Prefer `init`-only setters and `required` properties over
mutable setters to force valid, immutable construction. `required` members and
constructors are both acceptable ways to enforce mandatory initialization; prefer
`required`/`init` to avoid telescoping constructors, but use a constructor when you
need to enforce a cross-field invariant at construction time. (Be aware some
serializers need configuration to honor `required`/`init`-only members.)

```csharp
public record Person
{
    public required string FirstName { get; init; }
    public required string LastName { get; init; }
}
```

**`CS-IMM-03` (SHOULD)** Mark fields `readonly` when set only in the constructor;
prefer immutable collections (`ImmutableArray<T>`, `IReadOnlyList<T>`) for exposed
state (`GEN-PRIN-26`, `GEN-DEF-12`).

**Pros:** thread-safety, value equality, predictable code.
**Cons:** allocation on each "mutation" (`with`), unsuited to large frequently-
mutated buffers — use `Span<T>`/pooling there.

---

## 9. Strings

**`CS-STR-01` (SHOULD)** Use **string interpolation** (`$"{x}, {y}"`) for readable
concatenation of short strings (MS guidance), and **expression-based** (not
positional) interpolation.

**`CS-STR-02` (SHOULD)** Use **`StringBuilder`** for building strings in loops or
large concatenations — string is immutable, so `+=` in a loop is O(n²) allocation
(`GEN-PERF-04`).

**`CS-STR-03` (SHOULD)** Prefer **raw string literals** (`"""..."""`) over escape
sequences/verbatim strings for multi-line or quote-heavy content (MS guidance).

**`CS-STR-04` (SHOULD)** Specify `StringComparison` explicitly for comparisons:
`StringComparison.Ordinal`/`OrdinalIgnoreCase` for identifiers and machine
strings; `CurrentCulture`/`InvariantCulture` only for user-facing text. Culture-
unaware comparisons cause subtle locale bugs.

---

## 10. Collections & LINQ

**`CS-COL-01` (SHOULD)** Expose the least-specific useful type: return
`IReadOnlyList<T>`/`IEnumerable<T>` from APIs rather than `List<T>` to preserve
encapsulation; accept the most general type you can use.

**`CS-COL-02` (SHOULD)** Never return `null` for a collection — return an empty
collection (`[]`, `Array.Empty<T>()`, `Enumerable.Empty<T>()`) so callers can
iterate safely (`GEN-DEF-10`).

**`CS-COL-03` (SHOULD)** Use **LINQ** for readable collection queries; name query
variables meaningfully; put `Where` before other clauses to reduce the working set
(MS guidance). Use implicit typing for query results.

**`CS-COL-04` (SHOULD)** Be aware LINQ is **lazily evaluated** (deferred
execution). Don't enumerate the same query multiple times (re-runs the work);
materialize with `.ToList()`/`.ToArray()` when you need a stable, repeatedly-read
result. Beware `IQueryable` vs `IEnumerable` (DB vs in-memory) — an accidental
`AsEnumerable` can pull the whole table into memory.

**Pros of LINQ:** declarative, readable, composable.
**Cons of LINQ:** deferred execution surprises, allocation/closure overhead in hot
paths, and easy accidental multiple-enumeration or N+1 with EF Core — profile hot
paths (`GEN-PERF-01`).

---

## 11. Exceptions & error handling

Builds on [error handling](../03-error-handling.md).

**`CS-ERR-01` (SHOULD)** Throw the most specific built-in exception that fits
(`ArgumentNullException`, `ArgumentOutOfRangeException`, `InvalidOperationException`)
before creating custom types. Custom exceptions derive from `Exception` and end in
`Exception`.

**`CS-ERR-02` (MUST NOT)** Don't catch `System.Exception` (or `catch {}`) except at
a top-level boundary that logs and converts. Catch specific types, and use
**exception filters** (`catch (X e) when (...)`) to refine without unwinding. (MS
guidance.)

**`CS-ERR-03` (SHOULD)** Re-throw with `throw;` (preserves the stack trace), not
`throw ex;` (resets it). Chain causes via `new X("...", inner)`.

**`CS-ERR-04` (MUST NOT)** Don't use exceptions for control flow; don't throw from
hot paths for ordinary conditions. Prefer `Try...` patterns
(`int.TryParse`, `dict.TryGetValue`) that return success without throwing.

**`CS-ERR-05` (SHOULD)** For expected, frequent failures consider a Result type
(e.g. a discriminated-union-like `OneOf`/custom `Result<T>`) instead of exceptions
(`GEN-ERR-10`).

---

## 12. Resource management: IDisposable & using

**`CS-DISP-01` (MUST)** Wrap `IDisposable` resources in a `using` statement/
declaration so `Dispose()` runs on every path (including exceptions). Prefer the
brace-less `using` declaration for whole-scope lifetimes (MS guidance).

```csharp
using var connection = new SqlConnection(connectionString);
await connection.OpenAsync();
// disposed automatically at end of scope
```

**`CS-DISP-02` (SHOULD)** Implement the **dispose pattern** correctly when you own
unmanaged resources or disposable fields; implement `IAsyncDisposable` for async
cleanup (`await using`). Make `Dispose` idempotent and exception-safe.

**`CS-DISP-03` (SHOULD NOT)** Don't rely on finalizers for normal cleanup — they
run nondeterministically and hurt GC. Use them only as a safety net for unmanaged
resources, and suppress finalization in `Dispose` (`GC.SuppressFinalize`).

---

## 13. Async/await

Builds on [concurrency](../07-concurrency.md) §8.

**`CS-ASYNC-01` (SHOULD)** Use `async`/`await` for I/O-bound work; return `Task`/
`Task<T>` (or `ValueTask<T>` for hot paths that often complete synchronously).
Suffix async methods with `Async`.

**`CS-ASYNC-02` (MUST NOT)** **Never block on async code** (`.Result`, `.Wait()`,
`.GetAwaiter().GetResult()`) — it can deadlock (especially with a sync context) and
exhaust the thread pool. Be async all the way (`GEN-CONC-13`).

**`CS-ASYNC-03` (SHOULD)** **`async void` is forbidden** except for event handlers
— exceptions in `async void` can't be caught and crash the process. Return `Task`.

**`CS-ASYNC-04` (SHOULD)** In **library** code, use `ConfigureAwait(false)` on
awaits to avoid capturing/forcing the caller's synchronization context (prevents
deadlocks and improves performance). Application/UI code that needs the context
omits it. (MS guidance.) Note that ASP.NET Core has **no** synchronization context,
so `ConfigureAwait(false)` is a no-op there; follow the repository's existing
convention for consistency rather than adding it inconsistently.

**`CS-ASYNC-05` (SHOULD)** Accept and honor a `CancellationToken` on async APIs;
put timeouts on awaited operations. Avoid `Task.Run` to "fake" async over CPU work
in libraries (let the caller decide). Don't fire-and-forget Tasks without
observing exceptions.

---

## 13a. Threading & synchronization

Builds on [concurrency](../07-concurrency.md). C#-specific:

**`CS-THREAD-01` (SHOULD)** For simple critical sections use `lock` (`Monitor`); for
shared counters/flags use `Interlocked`; for shared collections prefer the
`System.Collections.Concurrent` types over manual locking.

**`CS-THREAD-02` (SHOULD NOT)** Don't `lock` on `this`, a `Type`, or an interned
`string` — these are publicly reachable and invite deadlocks. Lock on a private
`readonly object` (or a dedicated `Lock` instance on .NET 9+).

**`CS-THREAD-03` (MUST NOT)** Don't hold a `lock` across an `await` (the C# compiler
disallows `await` inside a `lock` body); use `SemaphoreSlim.WaitAsync` for async
mutual exclusion instead.

**`CS-THREAD-04` (SHOULD)** Bound parallelism and back-pressure (`Parallel`,
`Channel<T>`, dataflow); don't spawn unbounded `Task`s.

---

## 13b. Dependency injection

**`CS-DI-01` (SHOULD)** Inject dependencies through the constructor; depend on
interfaces/abstractions, not concrete types (`GEN-PRIN-11`). Register services with
the correct lifetime (singleton / scoped / transient) and never inject a shorter-
lived service into a longer-lived one (captive dependency).

**`CS-DI-02` (SHOULD)** Treat a constructor that needs **many** dependencies as a
design smell — it usually means the type has too many responsibilities (`GEN-PRIN-07`).
Split it before reaching for service-locator workarounds.

---

## 14. Delegates, events, lambdas

**`CS-DEL-01` (SHOULD)** Use `Func<>`/`Action<>` instead of declaring custom
delegate types in most cases (MS guidance).

**`CS-DEL-02` (SHOULD)** Use a lambda for an event handler you won't need to
detach; keep a method reference if you must unsubscribe (to avoid leaks — an
undetached handler keeps the subscriber alive).

**`CS-DEL-03` (SHOULD)** Beware closure captures in loops and in long-lived
delegates — captured variables can extend object lifetimes and capture the wrong
value; capture a local copy when needed.

---

## 15. Namespaces & usings

**`CS-NS-01` (SHOULD)** Use **file-scoped namespaces** (`namespace Foo.Bar;`) —
one namespace per file, less indentation (MS guidance).

**`CS-NS-02` (SHOULD)** Place `using` directives **outside** the namespace
declaration. Inside-the-namespace usings are context-sensitive and can resolve to
an unexpected nested namespace, silently changing which type is referenced (MS
guidance, with the `WaitUntil`/`Azure` example).

**`CS-NS-03` (CONSIDER)** Use `global using` (and implicit usings) for ubiquitous
namespaces to reduce boilerplate across the project.

---

## 16. API & library design

Per Framework Design Guidelines:

**`CS-API-01` (SHOULD)** Design for the caller: minimal, discoverable, hard-to-
misuse APIs. Prefer properties for simple data, methods for actions/expensive
operations. Don't expose mutable internal collections.

**`CS-API-02` (SHOULD)** Validate all public arguments and throw the appropriate
`Argument*Exception` with `nameof(param)`. Public APIs are a trust boundary
(`GEN-DEF-02`).

**`CS-API-03` (SHOULD)** Keep the public surface minimal (`internal`/`private` by
default) and bind it with SemVer (`GEN-PRIN-30`). Use `[Obsolete]` to deprecate
before removing.

**`CS-API-04` (SHOULD)** Prefer dependency injection (constructor injection) over
`static`/singletons/`new`-ing dependencies — improves testability and follows DIP
(`GEN-PRIN-11`). Use the built-in `Microsoft.Extensions.DependencyInjection` with
correct service lifetimes (singleton/scoped/transient); don't capture scoped
services in singletons.

---

## 17. Security

See [security](../04-security.md). C#-specific:

**`CS-SEC-01` (MUST)** Use parameterized queries / an ORM (EF Core, Dapper with
parameters); never string-concatenate SQL (`GEN-SEC-03`).
**`CS-SEC-02` (MUST)** Use `RandomNumberGenerator` (not `Random`) for security
tokens; `System.Security.Cryptography` for crypto; never `MD5`/`SHA1` for
passwords — use ASP.NET Core Identity / a vetted KDF (`GEN-SEC-10`, `GEN-SEC-22`).
**`CS-SEC-03` (MUST NOT)** Never use `BinaryFormatter` (removed/insecure) or
deserialize untrusted data into arbitrary types (`GEN-SEC-32`).
**`CS-SEC-04` (SHOULD)** Validate/encode output; rely on framework anti-XSS/anti-
CSRF (Razor auto-encodes; use `[ValidateAntiForgeryToken]`); store secrets in
user-secrets/Key Vault, not source.

---

## 18. Testing

See [testing](../05-testing.md). C#-specific:

**`CS-TEST-01` (SHOULD)** Use xUnit (or NUnit/MSTest), with `FluentAssertions` for
readable assertions and `Moq`/`NSubstitute` for mocking interfaces. Inject
dependencies so they're substitutable.
**`CS-TEST-02` (SHOULD)** Name tests descriptively (`Method_Scenario_Expected`),
follow AAA, and prefer testing through public behavior. Use `Theory`/`InlineData`
for parameterized cases.
**`CS-TEST-03` (CONSIDER)** Use `Testcontainers` for realistic integration tests
and `BenchmarkDotNet` for performance benchmarks (`GEN-PERF-06`).

---

## 19. Anti-patterns

- Runtime type names (`String`, `Int32`) instead of keywords (`string`, `int`).
- `var` where the type isn't obvious; explicit type omitted in `foreach`.
- Disabling/ignoring NRTs or scattering `!` to silence the compiler.
- `catch (Exception)` / empty catch; `throw ex;` resetting stack traces.
- Exceptions as control flow; not using `Try...` patterns.
- Blocking on async (`.Result`/`.Wait()`); `async void`; missing
  `ConfigureAwait(false)` in libraries; fire-and-forget tasks.
- Not disposing `IDisposable`; relying on finalizers.
- Returning `null` collections; exposing mutable internal lists.
- String `+=` in loops; culture-sensitive comparisons for machine strings.
- Multiple enumeration of LINQ queries; accidental `IEnumerable` over `IQueryable`.
- Static/singletons instead of DI; capturing scoped services in singletons;
  god-constructors with too many dependencies.
- `lock`ing on `this`/`Type`/`string`; holding a lock across `await`.
- Records (value equality) used for identity-rich domain entities.
- `BinaryFormatter`; concatenated SQL; `Random` for tokens.

---

## Quick checklist

- [ ] `.editorconfig` + analyzers + `TreatWarningsAsErrors`; `dotnet format` in CI.
- [ ] Nullable reference types enabled; intent expressed via `T?`/`T`; guard
      clauses with `ArgumentNullException.ThrowIfNull`.
- [ ] PascalCase/`I`-interface/`_camelCase`-private/`camelCase`-local naming; async
      methods suffixed `Async`.
- [ ] Allman braces, 4 spaces, all control blocks braced.
- [ ] Language keywords for built-in types; `int` over unsigned by default; classes
      `sealed` unless designed for inheritance.
- [ ] `var` only when type is obvious; explicit type in `foreach`.
- [ ] Records/`init`/`required`/`readonly`/immutable collections for data.
- [ ] Interpolated & raw string literals; `StringBuilder` in loops; explicit
      `StringComparison`.
- [ ] APIs return read-only/empty (never null) collections; LINQ not multiply
      enumerated.
- [ ] Specific exceptions + filters; no `catch(Exception)`/empty catch; `throw;`;
      `Try...` for expected failures.
- [ ] `using` for all `IDisposable`/`IAsyncDisposable`; correct dispose pattern; no
      finalizer reliance.
- [ ] Async all the way; no blocking; no `async void`; `ConfigureAwait(false)` in
      libs; cancellation + timeouts honored.
- [ ] Locks on private objects (not `this`/`Type`/`string`); no lock across `await`;
      `Interlocked`/concurrent collections for shared state; bounded parallelism.
- [ ] Records for value objects (not identity entities); guard clauses use
      `ArgumentException.ThrowIfNullOrEmpty` for strings.
- [ ] File-scoped namespaces; usings outside the namespace.
- [ ] Constructor DI with correct lifetimes (no captive deps, no god-constructors);
      minimal public surface bound by SemVer.
- [ ] Parameterized SQL; CSPRNG/KDF for crypto; no `BinaryFormatter`/untrusted
      deserialization; output encoded.
- [ ] xUnit + AAA + DI-based mocking; integration & benchmark coverage where
      warranted.

---

## References

- Microsoft, ".NET Coding Conventions (C#)" — https://learn.microsoft.com/dotnet/csharp/fundamentals/coding-style/coding-conventions
- Microsoft, "Framework Design Guidelines" — https://learn.microsoft.com/dotnet/standard/design-guidelines/
- Microsoft, "Best practices for exceptions" — https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions
- Microsoft, "Asynchronous programming" & "Async guidance" — https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/
- Stephen Cleary, "Async/Await Best Practices" — https://learn.microsoft.com/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming
- Microsoft, "Secure coding guidelines" — https://learn.microsoft.com/dotnet/standard/security/secure-coding-guidelines
- .NET Runtime coding style — https://github.com/dotnet/runtime/blob/main/docs/coding-guidelines/coding-style.md
