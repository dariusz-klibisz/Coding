# Java — Coding Standards & State-of-the-Art Practices

> **Scope:** Java 17+ (LTS) through current LTS, including records, sealed
> types, pattern matching for `switch`, and virtual threads (Project Loom).
> **Primary sources:** Google Java Style Guide, Oracle's Java Language
> Specification and Effective Java (Joshua Bloch), OpenJDK JEP documents for
> the modern-feature rules below.
> **Relationship to general docs:** extends [general docs](../00-index.md). Where a
> general rule and a Java rule conflict, the Java rule wins for Java.

**Rule ID prefix:** `JAVA`

---

## Table of contents

1. [Tooling & enforcement](#1-tooling--enforcement)
2. [Naming conventions](#2-naming-conventions)
3. [Formatting & layout](#3-formatting--layout)
4. [Language idioms & state-of-the-art features](#4-language-idioms--state-of-the-art-features)
5. [Type safety / static analysis](#5-type-safety--static-analysis)
6. [Error handling](#6-error-handling)
7. [Memory / resource management](#7-memory--resource-management)
8. [Concurrency](#8-concurrency)
9. [Security](#9-security)
10. [Testing](#10-testing)
11. [Anti-patterns](#11-anti-patterns)
12. [Quick checklist](#quick-checklist)
13. [References](#references)

---

## 1. Tooling & enforcement

**`JAVA-TOOL-01` (SHOULD)** Format with **google-java-format** or an
equivalent deterministic formatter (via Spotless in the build), and lint
with **Error Prone** (compile-time bug-pattern detection) and/or
**SpotBugs** (`GEN-TOOL-02`/`04`/`06`).

**`JAVA-TOOL-02` (SHOULD)** Build with **Gradle** or **Maven** and commit
the wrapper (`gradlew`/`mvnw`) so every contributor and CI use the exact
same build-tool version (`GEN-VCS-14`).

**`JAVA-TOOL-03` (SHOULD)** Enable `-Xlint:all` and treat new compiler
warnings as CI failures once the baseline is clean (`GEN-TOOL-07`/`08`).

---

## 2. Naming conventions

Per Google Java Style / standard Java convention:

| Construct | Convention | Example |
|-----------|------------|---------|
| Class, interface, enum, record | `UpperCamelCase` | `OrderService`, `Status` |
| Method, field, variable, parameter | `lowerCamelCase` | `calculateTotal`, `itemCount` |
| Constant (`static final`) | `UPPER_SNAKE_CASE` | `MAX_RETRIES` |
| Package | all-lowercase, reverse-domain | `com.example.orders` |
| Type parameter | single uppercase letter, or descriptive + suffix `T` | `T`, `KeyT` |

**`JAVA-NAME-01` (SHOULD)** Suffix custom exception classes with
`Exception` (`OrderNotFoundException`), not `Error` — reserve `Error` for
the `java.lang.Error` hierarchy (unrecoverable JVM-level failures), which
application code should not extend (`GEN-ERR-07`).

**`JAVA-NAME-02` (SHOULD)** Name boolean methods/fields as assertions
(`isValid`, `hasPermission`) — mirrors `GEN-PRIN-16`.

---

## 3. Formatting & layout

**`JAVA-FMT-01` (SHOULD)** 2-space (Google style) or 4-space (traditional
Java) indentation — pick one per project and let the formatter enforce it;
consistency matters more than which one.

**`JAVA-FMT-02` (SHOULD)** One top-level class per file, file name matching
the public class name exactly (compiler-enforced for `public` classes).

---

## 4. Language idioms & state-of-the-art features

**`JAVA-IDIOM-01` (SHOULD)** Use **records** (Java 16+) for immutable
data-carrier classes instead of a hand-written class with a constructor,
getters, `equals`/`hashCode`/`toString` — records generate all of that from
the component list, and their immutability is `GEN-PRIN-26` enforced by the
language.

```java
// Bad: 40 lines of boilerplate for a plain data holder
public final class Point {
    private final double x, y;
    public Point(double x, double y) { this.x = x; this.y = y; }
    public double getX() { return x; }
    public double getY() { return y; }
    // equals, hashCode, toString omitted for brevity but required...
}

// Good: equivalent behavior, generated
public record Point(double x, double y) {}
```

**`JAVA-IDIOM-02` (SHOULD)** Use **sealed interfaces/classes** (Java 17+)
with pattern-matching `switch` (Java 21+) to model a closed set of
alternatives — the compiler enforces exhaustiveness over a sealed
hierarchy's known permitted subtypes (`GEN-DEF-14`).

```java
sealed interface Shape permits Circle, Square {}
record Circle(double radius) implements Shape {}
record Square(double side) implements Shape {}

double area(Shape shape) {
    return switch (shape) {                 // exhaustive, no default needed
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Square s -> s.side() * s.side();
    };
}
```

**`JAVA-IDIOM-03` (SHOULD)** Use the Streams API (`stream()`, `map`,
`filter`, `collect`) for declarative collection transformations where it
reads at least as clearly as an explicit loop; don't force a stream
pipeline onto logic that's genuinely clearer as an imperative loop
(readability over cleverness, `GEN-PRIN-01`).

**`JAVA-IDIOM-04` (SHOULD)** Prefer composition and interface default
methods over deep class inheritance for shared behavior
(`GEN-PRIN-12`); use `final` on classes not designed for extension so the
compiler enforces the "closed for modification unless deliberately opened"
boundary (`GEN-PRIN-08` caveat).

**`JAVA-IDIOM-05` (SHOULD)** Use the builder pattern for constructors with
many optional parameters instead of telescoping constructors or a large
positional-parameter constructor (`GEN-PRIN-22`); Lombok's `@Builder` or a
hand-written static nested `Builder` class are both acceptable, applied
consistently.

---

## 5. Type safety / static analysis

**`JAVA-TYPE-01` (SHOULD)** Enable and heed `@Nullable`/`@NonNull`
annotations (JSpecify, or a framework's equivalent) checked by a static
analyzer (NullAway, Error Prone) — Java has no built-in null-safety in the
type system, so a static-analysis layer is the practical substitute
(`GEN-DEF-09`).

**`JAVA-TYPE-02` (SHOULD)** Avoid raw generic types (`List` instead of
`List<String>`) — they disable generic type checking entirely and produce
unchecked-conversion warnings; always parameterize.

**`JAVA-TYPE-03` (SHOULD)** Use `Optional<T>` for a method's return type
when "no result" is a legitimate outcome the caller must handle — never as
a field type or a method parameter type, where it adds ceremony without
the compile-time benefit (Optional's design intent is specifically
return-type absence-signaling, per its author's guidance).

---

## 6. Error handling

Builds on [error handling](../03-error-handling.md).

**`JAVA-ERR-01` (SHOULD)** Prefer **unchecked** exceptions
(`RuntimeException` subtypes) for most application error signaling; reserve
checked exceptions for errors a well-informed caller can genuinely be
expected to recover from at the call site — checked exceptions that don't
meet this bar mainly produce boilerplate `catch`/rethrow chains and
encourage swallowing to satisfy the compiler (Effective Java, Item 71).

**`JAVA-ERR-02` (MUST NOT)** Never catch `Exception`/`Throwable` and
discard it silently (`catch (Exception e) {}`) — catch the specific type,
or catch broadly only at a top-level boundary (a request filter, `main`)
that logs and converts (`GEN-ERR-03`, `GEN-ERR-11`).

**`JAVA-ERR-03` (SHOULD)** Chain the cause when wrapping an exception
(`throw new ServiceException("...", e)`, never dropping `e`) so the
original stack trace survives (`GEN-ERR-04`).

**`JAVA-ERR-04` (SHOULD NOT)** Don't use exceptions for ordinary control
flow (e.g. throwing to break out of nested loops on a hot path) —
exceptions capture a stack trace on construction, which is measurably
costly if used as a routine mechanism (`GEN-ERR-02`).

---

## 7. Memory / resource management

**`JAVA-RES-01` (MUST)** Use **try-with-resources** for every
`AutoCloseable` (streams, connections, locks-as-resource) so `close()` runs
on every exit path including an exception (`GEN-DEF-15`).

```java
// Good: closed on every path, including exceptions thrown mid-block
try (var in = Files.newInputStream(path)) {
    return in.readAllBytes();
}
```

**`JAVA-RES-02` (SHOULD)** Understand the JVM's generational garbage
collector well enough to reason about allocation-heavy hot paths
(`GEN-PERF-08`/`09`); reach for object pooling only where profiling shows
GC pressure is the actual bottleneck, not preemptively (`GEN-PERF-01`
measure first).

**`JAVA-RES-03` (SHOULD)** Prefer immutable objects (`final` fields, no
setters, records) for value types shared across threads — immutability
removes a whole class of synchronization need (`GEN-PRIN-26`,
`GEN-CONC-02`).

---

## 8. Concurrency

Builds on [concurrency](../07-concurrency.md).

**`JAVA-CONC-01` (SHOULD)** Use the `java.util.concurrent` package
(`ExecutorService`, `ConcurrentHashMap`, `CompletableFuture`,
`java.util.concurrent.atomic`) rather than hand-rolled `synchronized`/
`wait`/`notify` — the concurrent collections and executors encode the
correct memory-visibility guarantees so application code doesn't have to
(`GEN-CONC-10`).

**`JAVA-CONC-02` (SHOULD)** Use **virtual threads** (Java 21+, JEP 444) for
high-concurrency I/O-bound workloads (thread-per-request servers, blocking
client calls) instead of manually managing a bounded platform-thread pool —
virtual threads let simple, synchronous-looking blocking code scale to a
very large number of concurrent in-flight requests without the
thread-pool-sizing tuning platform threads require.

**`JAVA-CONC-03` (MUST)** Never call `Thread.stop()`/`Thread.suspend()`
(deprecated, unsafe — can leave shared objects in a corrupted state);
signal cooperative cancellation via `Thread.interrupt()` and have long-
running work check `Thread.currentThread().isInterrupted()` (`GEN-CONC-15`).

**`JAVA-CONC-04` (SHOULD)** Bound thread pools explicitly
(`Executors.newFixedThreadPool`, or a `ThreadPoolExecutor` with an explicit
queue and rejection policy) — an unbounded pool (`newCachedThreadPool`
under sustained load) can exhaust memory/OS threads (`GEN-CONC-17`).

---

## 9. Security

Builds on [security](../04-security.md). Java-specific:

**`JAVA-SEC-01` (MUST)** Use `PreparedStatement` with bound parameters for
SQL, never string-concatenated queries built from user input
(`GEN-SEC-03`).

**`JAVA-SEC-02` (MUST)** Never deserialize untrusted data with native Java
serialization (`ObjectInputStream.readObject`) — it is a well-documented
remote-code-execution vector via gadget chains; use a data-only format
(JSON, Protobuf) with a strict schema instead (`GEN-SEC-32`).

**`JAVA-SEC-03` (MUST)** Use `SecureRandom`, never `java.util.Random`, for
tokens, keys, and any security-relevant randomness (`GEN-SEC-22`).

**`JAVA-SEC-04` (SHOULD)** Run OWASP Dependency-Check or Snyk against the
project's dependency tree in CI; keep the build-tool's dependency-lock
mechanism (Gradle lockfiles, Maven's dependency management) in place for
reproducible, auditable builds (`GEN-SEC-34`/`35`).

---

## 10. Testing

Builds on [testing](../05-testing.md). Java-specific:

**`JAVA-TEST-01` (SHOULD)** Use **JUnit 5** with `@ParameterizedTest` for
case tables, and **Mockito** (or a hand-written fake, preferred per
`GEN-TEST-11`) for test doubles at owned architectural boundaries.

**`JAVA-TEST-02` (SHOULD)** Use **AssertJ**'s fluent assertions
(`assertThat(result).isEqualTo(...)`) over JUnit's plain `assertEquals` for
clearer failure messages on complex objects/collections.

**`JAVA-TEST-03` (CONSIDER)** Use **Testcontainers** for integration tests
against a real database/message broker in an ephemeral container, rather
than mocking the data layer entirely (`GEN-TEST-12`).

---

## 11. Anti-patterns

- Hand-written boilerplate data classes instead of `record`.
- Raw generic types (`List` without a type parameter).
- `catch (Exception e) {}` swallowing errors silently.
- Checked exceptions used pervasively for conditions that aren't genuinely
  caller-recoverable, producing boilerplate rethrow chains.
- Native Java serialization (`ObjectInputStream`) applied to untrusted data.
- `java.util.Random` used for tokens/security-sensitive values.
- String-concatenated SQL instead of `PreparedStatement`.
- Hand-rolled `synchronized`/`wait`/`notify` where
  `java.util.concurrent` already provides a correct, tested abstraction.
- `Thread.stop()`/`Thread.suspend()` used instead of cooperative
  interruption.
- Unbounded thread pools (`newCachedThreadPool`) under sustained load.
- `Optional` used as a field type or method parameter instead of a return
  type.

---

## Quick checklist

- [ ] Formatter (google-java-format/Spotless) + Error Prone/SpotBugs clean;
      `-Xlint:all` new-warnings-as-errors.
- [ ] `record`/sealed types + pattern-matching `switch` used for data
      holders and closed alternatives.
- [ ] No raw generic types; `Optional` used only as a return type.
- [ ] Unchecked exceptions preferred for most errors; checked exceptions
      reserved for genuinely caller-recoverable conditions.
- [ ] No swallowed exceptions; cause chained on wrap; exceptions not used
      for ordinary control flow.
- [ ] try-with-resources for every `AutoCloseable`.
- [ ] `java.util.concurrent` abstractions preferred over hand-rolled
      `synchronized`/`wait`/`notify`; virtual threads for high-concurrency
      I/O-bound workloads.
- [ ] Cooperative interruption used, never `Thread.stop()`/`suspend()`;
      thread pools explicitly bounded.
- [ ] Parameterized SQL; no native-Java deserialization of untrusted data;
      `SecureRandom` for security-relevant values.
- [ ] Dependency scanning (OWASP Dependency-Check/Snyk) in CI; lockfile/
      dependency management committed.
- [ ] JUnit 5 + AssertJ; Testcontainers for realistic integration tests.

---

## References

- Google Java Style Guide — https://google.github.io/styleguide/javaguide.html
- Joshua Bloch, *Effective Java*, 3rd ed.
- Java Language Specification — https://docs.oracle.com/javase/specs/
- OpenJDK JEP 395 (Records), JEP 409 (Sealed Classes), JEP 441 (Pattern
  Matching for switch), JEP 444 (Virtual Threads) — https://openjdk.org/jeps/0
- Oracle, "The Java Tutorials — Concurrency" — https://docs.oracle.com/javase/tutorial/essential/concurrency/
- Error Prone — https://errorprone.info/ · NullAway — https://github.com/uber/NullAway
- JUnit 5 — https://junit.org/junit5/ · AssertJ — https://assertj.github.io/doc/
- Testcontainers — https://testcontainers.com/
