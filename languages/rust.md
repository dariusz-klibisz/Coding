# Rust — Coding Standards & State-of-the-Art Practices

> **Scope:** Rust 2021+ edition for systems, application, and embedded
> (`no_std`) development.
> **Primary sources:** The Rust Book, the Rust API Guidelines, Rust RFC
> process, `clippy` lint documentation.
> **Relationship to general docs:** extends [general docs](../00-index.md). Where a
> general rule and a Rust rule conflict, the Rust rule wins for Rust. For
> firmware-level concerns shared with C (interrupts, MMIO, functional safety
> process), see [`languages/embedded-c.md`](embedded-c.md) — the concepts
> transfer even though Rust's ownership model changes how they're expressed.

**Rule ID prefix:** `RS`

---

## Table of contents

1. [Tooling & enforcement](#1-tooling--enforcement)
2. [Naming conventions](#2-naming-conventions)
3. [Formatting & layout](#3-formatting--layout)
4. [Language idioms & state-of-the-art features](#4-language-idioms--state-of-the-art-features)
5. [Type safety / static analysis](#5-type-safety--static-analysis)
6. [Error handling](#6-error-handling)
7. [Ownership, borrowing & memory management](#7-ownership-borrowing--memory-management)
8. [Concurrency / async](#8-concurrency--async)
9. [Security & `unsafe`](#9-security--unsafe)
10. [Testing](#10-testing)
11. [Embedded / `no_std` notes](#11-embedded--no_std-notes)
12. [Anti-patterns](#12-anti-patterns)
13. [Quick checklist](#quick-checklist)
14. [References](#references)

---

## 1. Tooling & enforcement

**`RS-TOOL-01` (MUST)** Format with `rustfmt` (default config, or a
committed `rustfmt.toml`) — not configurable per-taste in practice; run it
in CI (`cargo fmt --check`) (`GEN-TOOL-02`).

**`RS-TOOL-02` (MUST)** Run **Clippy** in CI (`cargo clippy -- -D warnings`)
— Clippy catches idiom violations and real bug patterns the compiler alone
does not flag (`GEN-TOOL-04`).

**`RS-TOOL-03` (SHOULD)** Pin the toolchain (`rust-toolchain.toml`) and
commit `Cargo.lock` for applications (binaries) for reproducible builds
(`GEN-VCS-14`); libraries typically don't commit the lockfile so downstream
resolution stays flexible.

**`RS-TOOL-04` (SHOULD)** Run `cargo audit`/`cargo deny` in CI to catch
known-vulnerable or disallowed-license dependencies (`GEN-SEC-34`).

---

## 2. Naming conventions

Per the Rust API Guidelines (RFC 430):

| Construct | Convention | Example |
|-----------|------------|---------|
| Type, trait, enum variant | `UpperCamelCase` | `HttpClient`, `Result`, `Ordering::Less` |
| Function, method, variable, module | `snake_case` | `parse_config`, `retry_count` |
| Constant, `static` | `SCREAMING_SNAKE_CASE` | `MAX_RETRIES` |
| Crate | `kebab-case` (directory)/`snake_case` (in code) | `my-crate` / `my_crate` |
| Type-conversion method prefixes | `as_`/`to_`/`into_` per cost | `as_str` (cheap view), `to_owned` (clone), `into_bytes` (consumes) |

**`RS-NAME-01` (SHOULD)** Follow the `as_`/`to_`/`into_` convention exactly:
`as_*` for a cheap, non-consuming reinterpretation; `to_*` for an expensive,
non-consuming conversion (typically a clone); `into_*` for a consuming
conversion. Misusing this convention misleads callers about cost and
ownership.

**`RS-NAME-02` (SHOULD)** Name a fallible constructor `try_new`/`try_from`
returning `Result`, reserving a bare `new` for a constructor that cannot
fail (Rust API Guidelines).

---

## 3. Formatting & layout

**`RS-FMT-01` (MUST)** 4-space indentation, rustfmt defaults for line width
and brace placement — let `rustfmt` own every formatting decision.

**`RS-FMT-02` (SHOULD)** Order items in a module conventionally: `use`
declarations, then constants/statics, then types, then `impl` blocks, then
free functions — `rustfmt` doesn't reorder items, so this is a team
convention to apply manually.

---

## 4. Language idioms & state-of-the-art features

**`RS-IDIOM-01` (SHOULD)** Use `enum` (algebraic data types) to model a
closed set of states, and exhaustive `match` (no catch-all `_` unless a
default is genuinely intended) — the compiler rejects a non-exhaustive
`match`, turning "forgot the new variant" into a compile error
(`GEN-DEF-14`).

```rust
enum LoadState<T> {
    Idle,
    Loading,
    Loaded(T),
    Failed(String),
}

fn describe(state: &LoadState<Data>) -> &'static str {
    match state {
        LoadState::Idle => "idle",
        LoadState::Loading => "loading",
        LoadState::Loaded(_) => "loaded",
        LoadState::Failed(_) => "failed",
    }   // compiler error if a variant is added and unhandled
}
```

**`RS-IDIOM-02` (SHOULD)** Use `Option<T>` for absence and `Result<T, E>`
for fallibility instead of sentinel values (`-1`, `null`-equivalents) — this
is `GEN-DEF-09` enforced by the compiler: there is no `null` in safe Rust,
so every "might not have a value" case must be spelled out in the type.

**`RS-IDIOM-03` (SHOULD)** Prefer iterator combinators (`map`, `filter`,
`fold`, `collect`) over manual index-based loops where they read at least as
clearly — they compile to code as fast as a hand-written loop (zero-cost
abstraction) and eliminate off-by-one/index-mutation bugs.

**`RS-IDIOM-04` (SHOULD)** Use the **newtype pattern** (`struct UserId(u64);`)
to give a primitive a distinct type, preventing accidental mixing of
semantically different values that share a representation
(`GEN-DEF-04` parse-don't-validate, mirrors `KT-TYPE-03`).

**`RS-IDIOM-05` (SHOULD)** Use `impl Trait`/generics with trait bounds for
function parameters accepting "anything that behaves like X", and reserve
`dyn Trait` (trait objects) for genuine runtime polymorphism (a heterogeneous
collection of different types behind one interface) — the two aren't
interchangeable defaults; static dispatch (generics) is zero-cost, dynamic
dispatch (`dyn`) has a vtable-call cost and different object-safety rules.

**`RS-IDIOM-06` (SHOULD)** Use the builder pattern or functional options
(mirrors `GO-IDIOM-04`) for constructors with many optional parameters,
rather than an ever-growing `new(a, b, c, d, e, ...)`.

---

## 5. Type safety / static analysis

**`RS-TYPE-01` (SHOULD)** Avoid `unwrap()`/`expect()` on `Option`/`Result` in
library and production application code except where the invariant is truly
guaranteed by prior code and `expect("reason")` documents *why* it's safe
— an unhandled `unwrap()` panics on the very input path this library expects
"absence" or "failure" to be modeled explicitly (`GEN-DEF-09`).

**`RS-TYPE-02` (SHOULD)** Enable `#![warn(clippy::all, clippy::pedantic)]`
selectively (not blanket, to avoid pedantic-lint fatigue) and treat new
warnings as CI failures once the baseline is clean (`GEN-TOOL-08`/`22`).

**`RS-TYPE-03` (CONSIDER)** Use `#[non_exhaustive]` on public enums/structs
you expect to grow variants/fields, so downstream crates are forced to
handle "some other variant" and a future addition isn't a breaking change
(`GEN-PRIN-32`).

---

## 6. Error handling

Builds on [error handling](../03-error-handling.md). Rust's `Result<T, E>`
is the canonical `GEN-ERR-10` "Result type" model, made ergonomic by the `?`
operator.

**`RS-ERR-01` (MUST)** Use `Result<T, E>` for fallible operations and
propagate with `?` rather than manual `match`-and-return boilerplate at
every call site.

```rust
fn load_config(path: &str) -> Result<Config, ConfigError> {
    let text = std::fs::read_to_string(path)
        .map_err(|e| ConfigError::Io { path: path.into(), source: e })?;
    parse(&text)
}
```

**`RS-ERR-02` (SHOULD)** Use `thiserror` for library error types (derives
`std::error::Error` with structured variants callers can match on) and
`anyhow`/`eyre` for application-level error aggregation where callers only
need to report/log the error, not match on its variant (`GEN-ERR-07`
"design the hierarchy around what the catcher needs").

**`RS-ERR-03` (MUST NOT)** Reserve `panic!` for genuinely unrecoverable
programmer errors and broken invariants — not for ordinary, input-driven
failure, which `Result` already models (`GEN-ERR-02`). A library function
processing external/untrusted input MUST NOT panic on malformed input;
return `Err`.

**`RS-ERR-04` (SHOULD)** Preserve the error chain: implement/derive `source()`
(via `thiserror`'s `#[source]`) so `anyhow`/logging can print the full
causal chain, not just the outermost message (`GEN-ERR-04`/`14`).

---

## 7. Ownership, borrowing & memory management

**`RS-RES-01` (MUST)** Let RAII (`Drop`) own resource cleanup — files,
locks, connections release deterministically when their owning value goes
out of scope, on every exit path including a panic unwind
(`GEN-DEF-15`). Don't fight this with manual cleanup calls the type already
does automatically.

**`RS-RES-02` (SHOULD)** Prefer borrowing (`&T`/`&mut T`) over cloning when
a function only needs temporary access — an unnecessary `.clone()` to
silence the borrow checker is usually a sign the function's ownership
design, not the checker, is wrong; fix the design rather than reaching for
`clone()` reflexively (`GEN-PERF-08`).

**`RS-RES-03` (SHOULD)** Use `Rc<RefCell<T>>`/`Arc<Mutex<T>>` deliberately
and sparingly — shared, interior-mutable ownership sidesteps the borrow
checker's compile-time guarantees in favor of a runtime check
(`RefCell` panics, `Mutex` blocks); reach for it only when genuine shared
ownership is required, not as a default escape hatch from lifetime
reasoning.

**`RS-RES-04` (SHOULD)** Return owned data or a borrowed slice with an
explicit lifetime rather than leaking implementation details through an
overly-generic lifetime parameter; let the compiler's borrow checker, not a
runtime check, prove memory safety wherever possible.

---

## 8. Concurrency / async

Builds on [concurrency](../07-concurrency.md). Rust's ownership model
enforces the "no shared mutable state without synchronization" rule
(`GEN-CONC-01`) *at compile time* via the `Send`/`Sync` marker traits.

**`RS-CONC-01` (MUST)** Trust `Send`/`Sync` bounds rather than working
around them (e.g. with `unsafe impl Send`) unless you have proven the type
is actually safe to share/transfer across threads — the compiler rejecting
a type as `!Send` is very often correctly identifying a real data race the
code hasn't handled yet.

**`RS-CONC-02` (SHOULD)** Use channels (`std::sync::mpsc`, or
`tokio::sync::mpsc` in async code) for message-passing designs, and
`Arc<Mutex<T>>`/`Arc<RwLock<T>>` for genuinely shared state — pick
deliberately per `GEN-CONC-02`'s preference order (no sharing → immutable →
safe abstraction → locks).

**`RS-CONC-03` (SHOULD)** Use `async`/`.await` (Tokio or async-std) for
I/O-bound concurrency; never call a blocking function directly inside an
async task without `spawn_blocking` (Tokio) — this is Rust's version of
`GEN-CONC-14`'s "never block the event loop."

**`RS-CONC-04` (SHOULD)** Propagate cancellation and timeouts explicitly
(`tokio::time::timeout`, a `CancellationToken`) — Rust's async model has no
built-in implicit cancellation-on-drop guarantee for arbitrary futures
beyond what the executor and the future's own `Drop` impl provide, so
design cancellation into the API rather than assuming it.

---

## 9. Security & `unsafe`

Builds on [security](../04-security.md).

**`RS-SEC-01` (MUST)** Treat every `unsafe` block as a proof obligation: it
must be as small as possible, and carry a `// SAFETY:` comment explaining
*why* the invariants `unsafe` requires (valid pointers, correct lifetimes,
no aliasing violations) actually hold at that call site.

```rust
// SAFETY: `ptr` was obtained from `Box::into_raw` immediately above and
// has not been freed or aliased since; the box being reconstructed here
// is the only owner.
let boxed = unsafe { Box::from_raw(ptr) };
```

**`RS-SEC-02` (SHOULD)** Prefer a safe abstraction (a well-reviewed crate,
or a safe wrapper type with an internal `unsafe` block) over scattering
`unsafe` through application code — concentrate the audit surface
(`GEN-DEF-16` barricade pattern applied to memory safety).

**`RS-SEC-03` (MUST)** Use parameterized queries (`sqlx`, `diesel`) for SQL,
never string-formatted queries with untrusted input (`GEN-SEC-03`); use
`rand::rngs::OsRng`/the `getrandom` crate (not `rand::thread_rng()`'s
default PRNG assumptions for security purposes) for cryptographic
randomness — verify the specific RNG type is documented as CSPRNG-backed
before using it for tokens/keys (`GEN-SEC-22`).

**`RS-SEC-04` (SHOULD)** Run `cargo audit`/`cargo deny` (§1) and `cargo geiger`
(reports `unsafe` usage across the dependency tree) to track the
project's total unsafe-code exposure, including transitively via
dependencies.

---

## 10. Testing

Builds on [testing](../05-testing.md). Rust-specific:

**`RS-TEST-01` (SHOULD)** Use built-in `#[test]` functions and
`#[cfg(test)] mod tests` co-located with the code under test for unit
tests; put integration tests in `tests/` (compiled as a separate crate,
exercising only the public API).

**`RS-TEST-02` (SHOULD)** Use **proptest** or **quickcheck** for
property-based testing of invariant-rich logic (parsers, serialization
round-trips) (`GEN-TEST-17`).

**`RS-TEST-03` (CONSIDER)** Use `cargo fuzz` (libFuzzer integration) for any
function parsing untrusted bytes — Rust's memory safety doesn't prevent
logic bugs, panics, or (in `unsafe` code) actual memory-safety violations
that fuzzing can still find (`GEN-TEST-18`).

**`RS-TEST-04` (SHOULD)** Use `#[should_panic]` sparingly and only for
genuinely-expected panics (a documented precondition violation) — prefer
asserting a returned `Err` for anything that is a normal, recoverable
failure mode.

---

## 11. Embedded / `no_std` notes

For firmware targets, builds on [embedded C](embedded-c.md) for the
hardware-level concepts (ISRs, MMIO, functional-safety process); Rust
changes how they're expressed, not whether they apply.

**`RS-EMB-01` (SHOULD)** Use `#![no_std]` with a chosen allocator (or
`heapless`'s fixed-capacity collections to avoid heap allocation entirely)
for microcontroller targets with no OS-provided heap; rely on the borrow
checker to prove absence of aliasing/use-after-free even without a
garbage collector (`GEN-PERF-09`).

**`RS-EMB-02` (SHOULD)** Use the `embedded-hal` trait ecosystem for
peripheral abstractions so driver code is portable across chips, and a
PAC/HAL crate (generated from the vendor's SVD file) for register access
instead of hand-rolled unsafe pointer arithmetic on memory-mapped I/O
addresses (mirrors `EC` §16 MMIO guidance, made safer by the type system).

**`RS-EMB-03` (SHOULD)** Wrap interrupt-shared state in an
interrupt-safe abstraction (`critical-section`, `cortex-m`'s `Mutex`
tied to a `CriticalSection` token) rather than a bare `static mut`, which
requires `unsafe` on every access and provides none of the aliasing
guarantees a proper abstraction gives (`EC-CONC` equivalent for Rust).

---

## 12. Anti-patterns

- `unwrap()`/`expect()` on a `Result`/`Option` whose failure is a normal,
  reachable runtime condition.
- `panic!` used for ordinary error signaling instead of `Result`.
- Reflexive `.clone()` to silence the borrow checker instead of fixing the
  ownership design.
- `unsafe` blocks with no `// SAFETY:` justification, or scattered broadly
  instead of concentrated in a reviewed abstraction.
- `unsafe impl Send`/`Sync` added to make a compile error disappear without
  proving the underlying data race is actually absent.
- String-formatted SQL instead of parameterized queries.
- Non-CSPRNG randomness used for tokens/keys.
- `Rc<RefCell<T>>`/`Arc<Mutex<T>>` reached for by default instead of
  reconsidering the ownership design.
- Non-exhaustive `match` with a catch-all `_` that silently swallows new
  enum variants that should have been handled explicitly.
- Blocking calls inside an async task without `spawn_blocking`.
- Bare `static mut` for interrupt-shared state on embedded targets.

---

## Quick checklist

- [ ] `cargo fmt --check` + `cargo clippy -- -D warnings` clean; toolchain
      pinned; `Cargo.lock` committed for binaries.
- [ ] `cargo audit`/`cargo deny` run in CI.
- [ ] `Option`/`Result` used for absence/fallibility; no unjustified
      `unwrap()`/`expect()` on reachable failure paths.
- [ ] Exhaustive `match` over enums; `#[non_exhaustive]` on public types
      expected to grow.
- [ ] Errors propagated with `?`; `thiserror` for library errors, `anyhow`/
      `eyre` for application aggregation; error chain preserved.
- [ ] `panic!` reserved for unrecoverable programmer errors, never for
      input-driven failure in a library's public API.
- [ ] Ownership/borrowing designed deliberately; `.clone()`/`Rc<RefCell<_>>`/
      `Arc<Mutex<_>>` used intentionally, not reflexively.
- [ ] Every `unsafe` block minimal with a `// SAFETY:` comment; concentrated
      in reviewed abstractions.
- [ ] `Send`/`Sync` violations fixed, not worked around with `unsafe impl`.
- [ ] Async code never blocks without `spawn_blocking`; cancellation/timeouts
      explicit.
- [ ] Parameterized SQL; CSPRNG-backed randomness for tokens/keys.
- [ ] Unit + integration tests; proptest/quickcheck for invariants; fuzzing
      for untrusted-byte parsers.
- [ ] Embedded: `no_std` + `heapless`/chosen allocator; `embedded-hal`
      abstractions; interrupt-safe state wrappers, not bare `static mut`.

---

## References

- *The Rust Programming Language* (the Book) — https://doc.rust-lang.org/book/
- Rust API Guidelines — https://rust-lang.github.io/api-guidelines/
- Rust RFC 430 (naming conventions) — https://rust-lang.github.io/rfcs/0430-finalizing-naming-conventions.html
- Clippy lints — https://rust-lang.github.io/rust-clippy/master/
- *Rust for Rustaceans* (Jon Gjengset) — advanced ownership/unsafe guidance.
- The Rustonomicon (`unsafe` Rust) — https://doc.rust-lang.org/nomicon/
- Tokio docs — https://tokio.rs/tokio/tutorial
- `embedded-hal` — https://github.com/rust-embedded/embedded-hal
- The Embedded Rust Book — https://docs.rust-embedded.org/book/
- cargo-audit — https://github.com/rustsec/rustsec
