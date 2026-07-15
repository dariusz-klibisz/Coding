# Defensive Programming & Robustness Techniques

> How to write code that behaves predictably in the presence of invalid inputs,
> unexpected states, and hostile environments — without over-engineering.

**Rule ID prefix:** `GEN-DEF`

---

## Table of contents

1. [What defensive programming is (and isn't)](#1-what-defensive-programming-is-and-isnt)
2. [Trade-offs: when to be defensive](#2-trade-offs-when-to-be-defensive)
3. [Design by Contract: preconditions, postconditions, invariants](#3-design-by-contract)
4. [Input validation at boundaries](#4-input-validation-at-boundaries)
5. [Assertions vs. exceptions](#5-assertions-vs-exceptions)
6. [Fail fast vs. fail safe](#6-fail-fast-vs-fail-safe)
7. [Guard clauses](#7-guard-clauses)
8. [Null / absence handling](#8-null--absence-handling)
9. [Defensive copying](#9-defensive-copying)
10. [Total functions & exhaustiveness](#10-total-functions--exhaustiveness)
11. [Resource safety](#11-resource-safety)
12. [The barricade / trust boundary pattern](#12-the-barricade--trust-boundary-pattern)
13. [Offensive programming (the complement)](#13-offensive-programming-the-complement)
14. [Anti-patterns](#14-anti-patterns)
15. [Quick checklist](#quick-checklist)
16. [References](#references)

---

## 1. What defensive programming is (and isn't)

**Definition:** Defensive programming is the practice of writing code that
continues to function correctly (or fails predictably and safely) under
unforeseen conditions — bad inputs, programmer mistakes in calling code, and
changing environments.

It is **not** the same as adding checks everywhere. Indiscriminate defensiveness
is itself an anti-pattern (see §2 and §14). The skill is knowing *where* a check
belongs and *what kind* of check it should be.

A useful mental model (from McConnell's *Code Complete*): code at the
**boundaries of your system / trust zone** should be paranoid about inputs; code
**inside** the trust zone, having already validated inputs at the boundary, can
assume its invariants hold and use assertions to catch programmer errors.

---

## 2. Trade-offs: when to be defensive

Defensive programming is a tool with real costs. Apply it deliberately.

**Pros**

- Bugs surface earlier and closer to their cause.
- Systems degrade predictably instead of corrupting data or crashing opaquely.
- Security: validation at boundaries blocks injection and malformed-input attacks.
- Better diagnostics: a precise "precondition X violated" beats a generic
  `NullPointerException` 10 frames away.

**Cons**

- **Code bloat & noise:** a check on every line obscures the real logic.
- **Performance:** redundant validation in hot paths or deep call stacks costs
  cycles (matters in embedded/real-time and tight loops).
- **Masking bugs:** the worst failure mode — silently "handling" an impossible
  condition (e.g. `catch` and continue) hides the defect instead of exposing it.
  Defensive code that returns a default for a state that "should never happen"
  can turn a loud crash into a silent data-corruption bug.
- **False security:** validating the same thing at every layer wastes effort and
  spreads the validation logic (a DRY violation).

**Guidance:**

| Context | Posture |
|---------|---------|
| Public API / trust boundary / external input | **Validate aggressively** (reject and report). |
| Internal function, inputs already validated | **Assert** programmer errors (cheap, disable-able), don't re-validate. |
| Hot loop / real-time / embedded ISR | Minimize runtime checks; rely on static analysis & invariants. |
| Security-sensitive parsing | Maximum validation; treat all input as hostile. |

> **Rule of thumb:** Validate untrusted data once, at the boundary, then trust it
> internally. Use assertions internally to catch *your own* mistakes, not to
> re-validate already-trusted data.

---

## 3. Design by Contract

**`GEN-DEF-01` (SHOULD)** Specify, for each non-trivial function, its
**preconditions** (what the caller must guarantee), **postconditions** (what the
function guarantees on return), and **invariants** (what stays true throughout an
object's life). Bertrand Meyer's Design by Contract (DbC).

**Why:** Contracts assign responsibility unambiguously. If a precondition is
violated, the *caller* has the bug; if a postcondition fails, the *callee* has
the bug. This eliminates the "defensive checks on both sides" duplication —
exactly one party is responsible for each guarantee.

```
def withdraw(account, amount):
    # Precondition: amount > 0 and amount <= account.balance (caller's job)
    assert amount > 0, "amount must be positive"
    assert amount <= account.balance, "insufficient funds"

    account.balance -= amount

    # Postcondition: balance never negative (callee's guarantee)
    assert account.balance >= 0
    return account.balance
    # Invariant (class-level): account.balance >= 0 at all observable times
```

**Pros**

- Clear ownership of correctness; no redundant double-checking.
- Contracts double as precise documentation and as test oracles.

**Cons**

- Requires discipline and a shared convention across the team.
- Runtime contract checks add overhead (often compiled out in release builds).

---

## 4. Input validation at boundaries

**`GEN-DEF-02` (MUST)** Validate all data crossing a trust boundary (network,
file, user input, IPC, environment variables, deserialization) before use.

**`GEN-DEF-03` (SHOULD)** Prefer an **allowlist** (accept only known-good) over a
**denylist** (reject known-bad). Denylists are perpetually incomplete.

**`GEN-DEF-04` (SHOULD)** Validate *and* normalize/canonicalize: validate type,
range, length, format, and encoding; then convert to a canonical internal
representation so the rest of the system sees uniform data.

**Why:** The boundary is the one place where you control whether bad data enters
the system. Validating there (and only there) keeps internal code clean and
prevents injection, overflow, and corruption. See
[03-security.md](04-security.md) for injection-specific guidance.

**Parse, don't validate:** Where the language allows, turn unstructured input
into a *typed* value once (parsing), so downstream code receives a value whose
validity is guaranteed by its type — rather than re-checking a raw string
everywhere.

```
# Weak: every consumer must remember to re-check
def send(email_str): ...

# Strong: parse once into a type that can't be invalid
@dataclass(frozen=True)
class Email:
    value: str
    def __post_init__(self):
        if "@" not in self.value:
            raise ValueError("invalid email")
def send(email: Email): ...  # downstream code trusts the type
```

---

## 5. Assertions vs. exceptions

This is the most commonly confused distinction in defensive programming.

| | **Assertion** | **Exception** |
|---|---|---|
| Catches | *Programmer* errors (bugs / "can't happen") | *Expected* runtime conditions (bad input, I/O failure) |
| Audience | The developer | The calling code / operator |
| Recoverable? | No — indicates a broken invariant | Yes — caller can handle |
| In production | Often compiled out / disabled | Always active |
| Example | "this list is non-empty here by construction" | "file not found", "user typed letters into a number field" |

**`GEN-DEF-05` (SHOULD)** Use **assertions** to document and verify conditions
that must be true if the code is correct. Use **exceptions/errors** for
conditions that can legitimately occur at runtime.

**`GEN-DEF-06` (MUST NOT)** Never put code with side effects inside an assertion,
and never use assertions to validate untrusted external input — assertions may be
disabled (e.g. Python `-O`, C `NDEBUG`), so a security check inside an assert can
vanish in production.

```
# WRONG: security/validation logic in an assertion (disappears under -O / NDEBUG)
assert user.is_authenticated, "must be logged in"   # NEVER

# WRONG: side effect in assertion
assert delete_temp_files()                          # NEVER

# RIGHT: assert an internal invariant
assert 0 <= index < len(buffer)                     # programmer-error guard
```

---

## 6. Fail fast vs. fail safe

These are two valid, opposite responses to an unexpected condition. Choose based
on the system's domain.

### Fail fast

Stop immediately when something is wrong, surfacing the error loudly.

**Pros:** Bugs found early and near their cause; no corrupt state propagates;
easy debugging.
**Cons:** A crash may be unacceptable in systems that must keep running.
**Use for:** development, internal services, anything where a wrong result is
worse than no result, detecting programmer errors.

### Fail safe / graceful degradation

Catch the problem and continue in a reduced but safe mode (default value, cached
result, disabled feature).

**Pros:** Availability and resilience; partial service beats total outage.
**Cons:** Can mask defects; risks operating on degraded/incorrect data; harder to
notice problems.
**Use for:** user-facing availability-critical systems, fault-tolerant/real-time
control where a controlled safe state is mandatory (e.g. an actuator must move to
a safe position, not halt).

> **Embedded/safety-critical note:** "Fail safe" has a precise meaning — on fault,
> transition to a defined *safe state*. This is often mandated by standards
> (IEC 61508, ISO 26262). See [languages/embedded-c.md](languages/embedded-c.md).

**`GEN-DEF-07` (SHOULD)** Decide the failure posture explicitly per subsystem and
document it. The dangerous default is "accidentally fail-safe" — swallowing
errors without realizing it.

---

## 7. Guard clauses

**`GEN-DEF-08` (SHOULD)** Handle preconditions and edge cases at the top of a
function with early returns/throws, keeping the main logic flat and un-nested.

**Why:** Guard clauses separate "is this even valid?" from "do the work,"
reducing nesting (cognitive complexity) and making the happy path obvious. See
`GEN-PRIN-25` in [00-principles.md](01-principles.md).

---

## 8. Null / absence handling

**`GEN-DEF-09` (SHOULD)** Avoid `null`/`nil`/`None` as an ordinary return value.
Prefer explicit absence types (`Optional`/`Maybe`/`Result`), empty collections,
or the Null Object pattern.

**Why:** Tony Hoare called the null reference his "billion-dollar mistake."
`null` is untyped absence: it passes type checks but explodes at use. Explicit
optional types force the caller to handle the empty case at compile time.

**`GEN-DEF-10` (SHOULD)** Never return `null` for a collection — return an empty
collection. Callers can iterate an empty collection safely; a `null` collection
forces a null check at every call site.

**`GEN-DEF-11` (SHOULD)** Enable and respect the language's null-safety features
(Kotlin nullables, C# nullable reference types, TS `strictNullChecks`,
`Optional`).

**Pros of explicit-absence types**

- Compiler enforces handling of the empty case.
- Intent is documented in the signature.

**Cons**

- Slightly more ceremony at call sites (mapping/unwrapping).
- Mixing with legacy null-returning code requires care at the seam.

---

## 9. Defensive copying

**`GEN-DEF-12` (SHOULD)** When accepting or returning a reference to mutable
internal state, copy it at the boundary so callers cannot mutate your internals
(and you cannot be affected by later mutation of caller-owned data).

```
# Bad: stores the caller's list; caller can mutate it behind our back
class Schedule:
    def __init__(self, dates):
        self._dates = dates          # aliased!

# Good: defensive copy on the way in (and out)
class Schedule:
    def __init__(self, dates):
        self._dates = list(dates)    # own copy
    @property
    def dates(self):
        return tuple(self._dates)    # immutable view out
```

**Pros:** Prevents action-at-a-distance bugs and protects invariants.
**Cons:** Allocation/copy cost (avoid in hot paths or for huge structures);
unnecessary if the data is already immutable — which is the cheaper solution
(see immutability, `GEN-PRIN-26`).

---

## 10. Total functions & exhaustiveness

**`GEN-DEF-13` (SHOULD)** Prefer **total** functions — defined for every possible
input — over **partial** functions that throw/UB on some inputs.

**`GEN-DEF-14` (SHOULD)** Make `switch`/`match` over enums and sum types
exhaustive, and let the compiler enforce it. When a new variant is added, the
compiler then flags every place that must handle it.

**Why:** Exhaustiveness turns "we forgot the new case" from a runtime bug into a
compile error. It is one of the cheapest, highest-value defensive techniques in
languages that support it (TS discriminated unions, C# switch expressions, Rust
`match`, etc.).

```
# Provoke a compile/type error if a new status is added but unhandled
def describe(status: Status) -> str:
    match status:
        case Status.ACTIVE:   return "active"
        case Status.PAUSED:   return "paused"
        case Status.CLOSED:   return "closed"
    # no default -> type checker flags missing variants
```

---

## 11. Resource safety

**`GEN-DEF-15` (MUST)** Guarantee that acquired resources (files, sockets, locks,
memory, handles) are released on *every* path, including exceptions and early
returns.

**Why:** Leaked resources cause gradual degradation (file-descriptor exhaustion,
memory growth, deadlock from unreleased locks) that is hard to diagnose because
the symptom appears far from the leak.

**Use the language's scope-bound mechanism:**

- C#: `using` / `await using`.
- Python: `with` (context managers).
- C++/Rust: RAII / destructors / `Drop`.
- C (embedded): `goto cleanup;` single-exit pattern, or explicit free on all
  branches.
- TS/JS: `try/finally`, or `using` (explicit resource management).

---

## 12. The barricade / trust boundary pattern

**`GEN-DEF-16` (SHOULD)** Establish an explicit "barricade": untrusted data is
validated and sanitized at the perimeter; everything inside the barricade treats
data as clean.

**Why:** This concentrates validation in one auditable place, avoids scattering
(and duplicating) checks, and lets interior code stay simple and fast. It also
makes the security review tractable — you audit the barricade, not every line.

```
   Untrusted zone            |  Barricade  |        Trusted zone
  (HTTP, files, env) -> [ validate+parse ] -> typed, clean domain objects
       paranoid checks here          assertions / no re-validation here
```

---

## 13. Offensive programming (the complement)

**`GEN-DEF-17` (CONSIDER)** During development, *amplify* failures rather than
hide them: crash on "impossible" conditions, fill freed memory with poison
values, abort on contract violations. In production, switch these to controlled
handling.

**Why:** Offensive programming ensures bugs are loud and unmissable while you can
still fix them cheaply, instead of being silently swallowed and shipped.
Defensive (tolerant) and offensive (intolerant) programming are complementary:
be intolerant of *programmer* errors, tolerant of *user/environment* errors.

---

## 14. Anti-patterns

- **Defensive everywhere / paranoid soup:** validating already-trusted internal
  data at every layer. Bloats code, hurts performance, duplicates logic.
- **Swallowing exceptions:** `try { ... } catch { /* ignore */ }` hides defects.
  At minimum log; usually re-throw or handle meaningfully. See
  [02-error-handling.md](03-error-handling.md).
- **Returning a default for "impossible" states:** converts a loud bug into a
  silent data-corruption bug. Prefer asserting/throwing on truly impossible
  states.
- **Validation in assertions:** disappears when assertions are disabled
  (`GEN-DEF-06`).
- **Null-returning collections:** forces null checks everywhere (`GEN-DEF-10`).
- **Catch-all at the wrong layer:** broad `catch (Exception)` deep in the stack
  prevents higher layers from making informed recovery decisions.

---

## Quick checklist

- [ ] Untrusted input validated **once** at the trust boundary (allowlist,
      type/range/length/format/encoding), then trusted internally.
- [ ] "Parse, don't validate" — input converted to typed domain objects early.
- [ ] Contracts (pre/post/invariant) defined for non-trivial functions; one party
      owns each guarantee (no double-checking).
- [ ] Assertions used for programmer errors; exceptions for runtime conditions.
- [ ] No validation/side effects inside assertions.
- [ ] Failure posture (fail-fast vs fail-safe) chosen and documented per subsystem.
- [ ] Guard clauses keep the happy path flat.
- [ ] Absence modeled with Optional/Result; collections never returned as null.
- [ ] Null-safety compiler features enabled.
- [ ] Mutable state defensively copied at boundaries (or made immutable).
- [ ] Enum/union handling is exhaustive and compiler-checked.
- [ ] Resources released on every path via scope-bound mechanisms.
- [ ] No swallowed exceptions; no silent defaults for impossible states.

---

## References

- Steve McConnell, *Code Complete*, 2nd ed., ch. 8 "Defensive Programming."
- Bertrand Meyer, *Object-Oriented Software Construction* (Design by Contract).
- Alexis King, "Parse, Don't Validate" — https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/
- Tony Hoare, "Null References: The Billion Dollar Mistake."
- SEI CERT C Coding Standard (validation, ERR, MEM) — https://wiki.sei.cmu.edu/confluence/display/c/
- OWASP Input Validation Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html
