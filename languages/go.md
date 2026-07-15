# Go — Coding Standards & State-of-the-Art Practices

> **Scope:** Go 1.21+ (generics, `slices`/`maps` stdlib packages, structured
> `log/slog`) for services, CLIs, and libraries.
> **Primary sources:** Effective Go, Go Code Review Comments, the Go
> specification, `golang.org/x/tools`/`staticcheck` conventions.
> **Relationship to general docs:** extends [general docs](../00-index.md). Where a
> general rule and a Go rule conflict, the Go rule wins for Go.

**Rule ID prefix:** `GO`

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

**`GO-TOOL-01` (MUST)** Format with `gofmt`/`goimports` — Go's formatting is
not configurable and not debated; unformatted code is a CI failure
(`GEN-TOOL-02`).

**`GO-TOOL-02` (SHOULD)** Run `go vet` and **staticcheck** in CI; both catch
real bug classes (`vet`: suspicious constructs like `Printf` format
mismatches; staticcheck: a much broader correctness/style rule set).

**`GO-TOOL-03` (SHOULD)** Run the race detector (`go test -race`, or
`go run -race` for manual checks) routinely, not just when a race is
suspected (`GEN-CONC-18`).

**`GO-TOOL-04` (SHOULD)** Pin the toolchain version in `go.mod` (`go 1.2x`)
and use `go mod tidy`/lockfile-equivalent verification in CI so builds are
reproducible.

---

## 2. Naming conventions

| Construct | Convention | Example |
|-----------|------------|---------|
| Exported identifier | `UpperCamelCase` (capitalized = exported) | `NewClient`, `ErrNotFound` |
| Unexported identifier | `lowerCamelCase` | `parseConfig`, `retryCount` |
| Package name | short, lowercase, no underscores | `http`, `json`, not `http_utils` |
| Interface with one method | method name + `-er` | `Reader`, `Stringer` |
| Error variable | `Err`-prefixed | `ErrNotFound`, `ErrTimeout` |
| Acronym | consistent casing, not mixed | `URL`/`ID` not `Url`/`Id` |

**`GO-NAME-01` (SHOULD)** Keep names short in small scopes (`i`, `buf`, `err`)
and longer/more descriptive as scope widens — a package-level exported name
carries no surrounding context, so it must be self-explanatory (Go Code
Review Comments; mirrors `GEN-PRIN-19`).

**`GO-NAME-02` (SHOULD)** Don't stutter the package name in an exported
identifier — `http.Client`, not `http.HTTPClient` — callers already qualify
with the package name.

**`GO-NAME-03` (SHOULD)** Name the receiver consistently and briefly (1–2
letters reflecting the type, e.g. `c *Client`), the same abbreviation across
every method of that type.

---

## 3. Formatting & layout

**`GO-FMT-01` (MUST)** Tabs for indentation (gofmt's default; do not
override). Braces on the same line (`if x {`) — Go's grammar requires this
for automatic semicolon insertion to work correctly.

**`GO-FMT-02` (SHOULD)** Group and order imports in gofmt/goimports'
standard blocks: standard library, then third-party, blank-line separated;
let the tool do this rather than hand-ordering.

**`GO-FMT-03` (SHOULD)** Keep the "happy path" unindented: return early on
error, don't wrap the success path in an `else` (Go Code Review Comments'
canonical guidance, same rationale as `GEN-PRIN-25`).

---

## 4. Language idioms & state-of-the-art features

**`GO-IDIOM-01` (SHOULD)** Accept interfaces, return concrete types — a
function's parameters should be the narrowest interface that satisfies its
needs (easy to fake in tests, decouples from a specific implementation);
its return type should be concrete (gives callers the full API, not a
narrowed view) unless multiple implementations genuinely exist.

**`GO-IDIOM-02` (SHOULD)** Keep interfaces small — the standard library's
one-method interfaces (`io.Reader`, `io.Writer`) are the idiomatic model.
"The bigger the interface, the weaker the abstraction" (Rob Pike) — mirrors
Interface Segregation (`GEN-PRIN-10`).

**`GO-IDIOM-03` (SHOULD)** Use generics (Go 1.18+) for genuinely
type-parametric algorithms/containers (a generic `Set[T]`, a `Map`/`Filter`
helper) — not as a default replacement for `interface{}`/`any` or as a way
to avoid writing two similar concrete functions when the duplication is
still small (`GEN-PRIN-04` Rule of Three).

```go
func Map[T, U any](in []T, f func(T) U) []U {
    out := make([]U, len(in))
    for i, v := range in {
        out[i] = f(v)
    }
    return out
}
```

**`GO-IDIOM-04` (SHOULD)** Use the functional-options pattern for
constructors with many optional parameters, instead of a large positional
parameter list or a config struct with unclear zero-value semantics
(`GEN-PRIN-22`).

```go
type Server struct{ timeout time.Duration; maxConns int }
type Option func(*Server)

func WithTimeout(d time.Duration) Option { return func(s *Server) { s.timeout = d } }

func NewServer(opts ...Option) *Server {
    s := &Server{timeout: 30 * time.Second, maxConns: 100} // sensible defaults
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

**`GO-IDIOM-05` (SHOULD)** Use zero values deliberately — design types so
their zero value (`var t T`) is immediately useful (`sync.Mutex`,
`bytes.Buffer`) rather than requiring an explicit constructor before use,
where that's practical.

---

## 5. Type safety / static analysis

**`GO-TYPE-01` (SHOULD)** Avoid `interface{}`/`any` in exported signatures
except at genuine type-erasure boundaries (generic containers, JSON
interop); prefer a concrete type or a generic type parameter so the
compiler keeps checking the call site (mirrors `SWIFT-TYPE-02`).

**`GO-TYPE-02` (SHOULD)** Use `staticcheck`'s nilness and struct-tag checks
in CI — Go has no compile-time null-safety, so static analysis is the
primary defense against nil-pointer classes of bugs that other languages in
this library catch via the type system.

---

## 6. Error handling

Builds on [error handling](../03-error-handling.md). Go's `error` values
are the dominant model in this ecosystem — see `GEN-ERR` §4 for the
error-code model this specializes.

**`GO-ERR-01` (MUST)** Check every returned `error` — never discard it with
`_` unless the call's failure mode is genuinely irrelevant and that's
documented at the call site (`GEN-ERR-08`).

```go
// WRONG — silently ignores a possible failure
data, _ := os.ReadFile(path)

// RIGHT — checked, wrapped with context
data, err := os.ReadFile(path)
if err != nil {
    return fmt.Errorf("reading config %s: %w", path, err)
}
```

**`GO-ERR-02` (SHOULD)** Wrap errors with `fmt.Errorf("...: %w", err)` to
preserve the chain (`GEN-ERR-04`/`14`) and let callers unwrap/match with
`errors.Is`/`errors.As` instead of string-matching the error message.

**`GO-ERR-03` (SHOULD)** Define sentinel errors (`var ErrNotFound = errors.New(...)`)
for conditions callers need to distinguish, and custom error types (a struct
implementing `Error() string`) when the error needs to carry structured
data — design the hierarchy around what the *catcher* needs (`GEN-ERR-07`).

**`GO-ERR-04` (SHOULD NOT)** Reserve `panic` for truly unrecoverable
programmer errors (a broken invariant, a startup-time misconfiguration) —
not for ordinary error signaling, which the `error` return value already
handles (`GEN-ERR-02`). A library MUST NOT panic across its public API
boundary for an input-driven condition; return an error instead.

**`GO-ERR-05` (SHOULD)** Use `defer`+`recover` only at a well-defined
boundary (a goroutine's entry point, an HTTP middleware) to convert a panic
into a logged error/5xx response — not as routine control flow.

---

## 7. Memory / resource management

**`GO-RES-01` (MUST)** Close every resource that needs closing
(`io.Closer`: files, network connections, `sql.Rows`) with `defer
resource.Close()` immediately after the successful open, so it runs on
every exit path including a panic (`GEN-DEF-15`).

```go
f, err := os.Open(path)
if err != nil {
    return err
}
defer f.Close()   // released on every return/panic below this line
```

**`GO-RES-02` (SHOULD)** Check (and usually log) the error returned by
`Close()` in `defer` when it's meaningful (a flush failure on write-close),
rather than a bare `defer f.Close()` that silently discards it, for
resources where a close failure is actionable.

**`GO-RES-03` (SHOULD)** Preallocate slices with `make([]T, 0, n)` when the
final size is known/estimable, to avoid repeated reallocation during append
growth (`GEN-PERF-08`).

---

## 8. Concurrency

Builds on [concurrency](../07-concurrency.md). Go's concurrency primitives
(goroutines, channels) are first-class language features.

**`GO-CONC-01` (SHOULD)** "Do not communicate by sharing memory; instead,
share memory by communicating" (Go proverb, `GEN-CONC-11`) — prefer channels
for handing off ownership of data between goroutines over shared state
guarded by a mutex, where the design fits naturally.

**`GO-CONC-02` (MUST)** Guard genuinely shared mutable state (a cache, a
counter accessed from multiple goroutines) with a `sync.Mutex`/`sync.RWMutex`
or an atomic type from `sync/atomic` — never leave concurrent map/slice
access unsynchronized (`GEN-CONC-04`); Go's built-in `map` is not
concurrency-safe.

**`GO-CONC-03` (MUST)** Never start a goroutine without a defined way for it
to stop — every long-running goroutine should observe a `context.Context`'s
cancellation or a dedicated `done` channel; a goroutine with no exit
condition is a leak, and leaked goroutines are Go's most common concurrency
resource bug (`GEN-CONC-17` bounded concurrency).

```go
func worker(ctx context.Context, jobs <-chan Job) {
    for {
        select {
        case <-ctx.Done():
            return                 // exits when the context is cancelled
        case job := <-jobs:
            process(job)
        }
    }
}
```

**`GO-CONC-04` (SHOULD)** Thread `context.Context` as the first parameter of
any function that does I/O or may block, and respect its deadline/
cancellation (`ctx.Done()`) — this is Go's idiomatic mechanism for the
timeout/cancellation propagation `GEN-ERR-19`/`GEN-CONC-15` require.

**`GO-CONC-05` (SHOULD)** Use `sync.WaitGroup` or `errgroup.Group` (from
`golang.org/x/sync/errgroup`) to wait for a bounded set of goroutines and
propagate the first error, rather than ad-hoc channel-based joining.

**`GO-CONC-06` (SHOULD)** Bound goroutine fan-out with a worker pool or a
buffered semaphore channel — an unbounded `go func()` per incoming request/
item can exhaust memory/OS threads under load (`GEN-CONC-17`).

---

## 9. Security

Builds on [security](../04-security.md). Go-specific:

**`GO-SEC-01` (MUST)** Use `database/sql` parameterized queries
(`db.Query(query, arg)`), never `fmt.Sprintf` string-built SQL with
untrusted input (`GEN-SEC-03`).

**`GO-SEC-02` (MUST)** Use `html/template` (auto-escaping) for any HTML
output containing untrusted data — never `text/template` for HTML responses,
and never string-concatenate untrusted input into a response body
(`GEN-SEC-08`).

**`GO-SEC-03` (MUST)** Use `crypto/rand`, never `math/rand`, for tokens,
keys, and any security-relevant randomness (`GEN-SEC-22`); note `math/rand`
is not cryptographically secure even though its API looks similar.

**`GO-SEC-04` (SHOULD)** Run `govulncheck` in CI to scan for known
vulnerabilities in the module's dependency graph, matched against actual
call paths (lower false-positive rate than a pure version-matching scanner)
(`GEN-SEC-34`).

---

## 10. Testing

Builds on [testing](../05-testing.md). Go-specific:

**`GO-TEST-01` (SHOULD)** Use the standard `testing` package with
table-driven tests (a slice of input/expected-output cases run in a loop)
as the default pattern — it's idiomatic, requires no framework, and keeps
each case's intent visible.

```go
func TestParsePort(t *testing.T) {
    cases := []struct{ in string; want int; wantErr bool }{
        {"80", 80, false},
        {"0", 0, true},
        {"not-a-number", 0, true},
    }
    for _, c := range cases {
        got, err := ParsePort(c.in)
        if (err != nil) != c.wantErr || got != c.want {
            t.Errorf("ParsePort(%q) = %d, %v; want %d, err=%v", c.in, got, err, c.want, c.wantErr)
        }
    }
}
```

**`GO-TEST-02` (SHOULD)** Use `t.Parallel()` for independent test cases to
speed up the suite, and `t.Cleanup()` for teardown instead of manual
`defer` scattered through the test body.

**`GO-TEST-03` (SHOULD)** Use the standard library's `httptest` for HTTP
handler tests and small, focused interfaces (§4) to substitute fakes for
external dependencies — Go's convention favors hand-written fakes over
mocking frameworks/codegen for most cases.

**`GO-TEST-04` (CONSIDER)** Use Go's built-in fuzzing (`go test -fuzz`,
1.18+) for parsers and any function processing untrusted byte input
(`GEN-TEST-18`).

---

## 11. Anti-patterns

- Discarding a returned `error` with `_` without a documented reason.
- `panic` used for ordinary error signaling instead of returning `error`.
- String-built SQL (`fmt.Sprintf` into a query) instead of parameterized
  queries.
- `math/rand` used for tokens/keys instead of `crypto/rand`.
- Concurrent map/slice access with no mutex/atomic guard.
- A goroutine started with no cancellation path (context/done channel) —
  a goroutine leak.
- Unbounded `go func()` fan-out per request/item with no worker pool/limit.
- Large, many-method interfaces instead of small, focused ones.
- `interface{}`/`any` used routinely in exported signatures instead of a
  concrete type or generic parameter.
- A resource opened without an immediately-following `defer Close()`.
- Package name stuttering in exported identifiers (`http.HTTPClient`).

---

## Quick checklist

- [ ] gofmt/goimports clean; `go vet` + staticcheck + `go test -race` in CI.
- [ ] Exported/unexported casing correct; no package-name stutter; small,
      consistent receiver names.
- [ ] Interfaces accepted, concrete types returned; interfaces kept small.
- [ ] Generics used for genuinely type-parametric code, not as a default
      `any` replacement.
- [ ] Every returned `error` checked; wrapped with `%w` and context; sentinel/
      custom error types match what callers need to distinguish.
- [ ] `panic` reserved for unrecoverable programmer errors; no panic across
      a library's public API for input-driven conditions.
- [ ] Every `io.Closer` closed via `defer` right after a successful open.
- [ ] Shared mutable state guarded by mutex/atomic; every goroutine has a
      cancellation path (context/done channel); fan-out bounded.
- [ ] `context.Context` threaded through I/O-bound calls; deadlines/
      cancellation respected.
- [ ] Parameterized SQL; `html/template` for HTML output; `crypto/rand` for
      security-relevant randomness; `govulncheck` in CI.
- [ ] Table-driven tests; `t.Parallel()`/`t.Cleanup()`; `httptest` for HTTP
      handlers; fuzzing for parsers.

---

## References

- Effective Go — https://go.dev/doc/effective_go
- Go Code Review Comments — https://go.dev/wiki/CodeReviewComments
- The Go Programming Language Specification — https://go.dev/ref/spec
- Go Blog, "Error handling and Go" — https://go.dev/blog/error-handling-and-go
- Go Blog, "Working with Errors in Go 1.13" (`%w`, `errors.Is`/`As`) — https://go.dev/blog/go1.13-errors
- Go Blog, "Share Memory By Communicating" — https://go.dev/blog/codelab-share
- Go proverbs — https://go-proverbs.github.io/
- staticcheck — https://staticcheck.io/
- govulncheck — https://go.dev/doc/tutorial/govulncheck
