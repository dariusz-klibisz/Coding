# Core Design Principles (Language-Agnostic)

> Foundational principles that apply to all languages. Language-specific files
> specialize these; when in conflict, the language-specific guidance wins for
> that language.

**Rule ID prefix:** `GEN-PRIN`

---

## Table of contents

1. [Readability first](#1-readability-first)
2. [KISS — Keep It Simple](#2-kiss--keep-it-simple)
3. [YAGNI — You Aren't Gonna Need It](#3-yagni--you-arent-gonna-need-it)
4. [DRY — Don't Repeat Yourself](#4-dry--dont-repeat-yourself)
5. [Separation of Concerns](#5-separation-of-concerns)
6. [Cohesion and Coupling](#6-cohesion-and-coupling)
7. [SOLID](#7-solid)
8. [Composition over Inheritance](#8-composition-over-inheritance)
9. [Law of Demeter](#9-law-of-demeter-principle-of-least-knowledge)
10. [Encapsulation & Information Hiding](#10-encapsulation--information-hiding)
11. [Principle of Least Astonishment](#11-principle-of-least-astonishment)
12. [Naming](#12-naming)
13. [Function & Module Design](#13-function--module-design)
14. [Immutability by Default](#14-immutability-by-default)
15. [Fail Fast / Make Illegal States Unrepresentable](#15-fail-fast--make-illegal-states-unrepresentable)
16. [Abstraction & Leaky Abstractions](#16-abstraction--leaky-abstractions)
17. [Semantic Versioning & API stability](#17-semantic-versioning--api-stability)
18. [Technical debt](#18-technical-debt)
19. [Quick checklist](#quick-checklist)
20. [References](#references)

---

## 1. Readability first

**`GEN-PRIN-01` (SHOULD)** Optimize code for reading, not writing. Code is read
far more often than it is written; a function written once may be read hundreds
of times during its lifetime.

**Why:** Maintenance dominates the software lifecycle cost. The bottleneck in
most teams is human comprehension, not typing speed or machine performance.
"Readability counts" is the explicit first principle of the Zen of Python and a
recurring theme across every major style guide.

**Practical implications:**

- Prefer clear over clever. A clever one-liner that takes five minutes to
  understand is a liability.
- Make the common path obvious and linear; push edge cases to the periphery.
- Consistency within a file/module beats global consistency, which beats
  personal preference.

---

## 2. KISS — Keep It Simple

**`GEN-PRIN-02` (SHOULD)** Choose the simplest design that satisfies the current,
known requirements.

**Why:** Complexity is the primary source of defects and the primary obstacle to
change. Every added abstraction, layer, or configuration knob is a permanent
cognitive tax and a potential failure mode.

**Pros of aggressively pursuing simplicity**

- Fewer defects (less code, fewer interactions).
- Faster onboarding and review.
- Easier to delete or change later.

**Cons / risks**

- Can be mistaken for under-engineering if genuine future needs are ignored.
- "Simple" is subjective; a simple implementation may push complexity onto
  callers. Aim for the design that minimizes *total* system complexity, not just
  local line count.

---

## 3. YAGNI — You Aren't Gonna Need It

**`GEN-PRIN-03` (SHOULD)** Do not build functionality, generality, or
configurability until there is a concrete, present need.

**Why:** Speculative generality is a leading cause of wasted effort and
accidental complexity. Predictions about future requirements are usually wrong,
and the speculative code still has to be maintained, tested, and understood in
the meantime.

**When to deviate (CONSIDER building ahead):**

- The cost of retrofitting later is genuinely high and irreversible (e.g. a
  public API contract, an on-disk data format, a security boundary).
- A trivial seam now prevents an expensive rewrite later.

**Pros**

- Less code, less maintenance, faster delivery.
- Designs stay grounded in real requirements.

**Cons**

- Occasionally requires refactoring when a need does materialize.
- Hard boundaries (APIs, schemas, protocols) genuinely benefit from forethought,
  so YAGNI must be balanced with [SemVer/API stability](#17-semantic-versioning--api-stability).

---

## 4. DRY — Don't Repeat Yourself

**`GEN-PRIN-04` (SHOULD)** Every piece of *knowledge* should have a single,
authoritative representation in the system.

**Why:** Duplicated knowledge drifts out of sync. When a rule changes, every
copy must be found and updated; missed copies become bugs.

**Critical nuance — DRY is about knowledge, not about text.** Two pieces of code
that look identical but represent *different* concepts that merely happen to
coincide today should **not** be merged. Coupling unrelated concepts because
they share code today creates a worse problem: a change to one concept forces a
change to the other.

```
# Bad (false DRY): forcing two unrelated rules through one function
def validate(value):  # used for both usernames AND product codes
    return 3 <= len(value) <= 20
# When product codes later allow length 50, username validation breaks.
```

**Pros**

- Single source of truth; changes apply everywhere at once.
- Smaller codebase.

**Cons / failure modes**

- Over-applied DRY creates tight coupling and the "wrong abstraction," which
  Sandi Metz famously argues is more costly than duplication. **A little
  duplication is cheaper than the wrong abstraction.**
- Prefer the **Rule of Three**: tolerate duplication until the same knowledge
  appears a third time, by which point the true abstraction is usually clear.

---

## 5. Separation of Concerns

**`GEN-PRIN-05` (SHOULD)** Divide a system into parts that each address a
distinct concern (e.g. presentation, business logic, persistence, transport).

**Why:** Concerns that change for different reasons and at different rates should
be isolated so a change to one does not ripple into the others. This is the
architectural expression of cohesion/coupling and of the Single Responsibility
Principle.

**Examples:** MVC/MVVM, layered/hexagonal architecture, separating I/O from pure
computation (which also dramatically improves testability).

---

## 6. Cohesion and Coupling

**`GEN-PRIN-06` (SHOULD)** Maximize cohesion *within* a module; minimize coupling
*between* modules.

- **Cohesion** = how closely the responsibilities of a single module relate to
  each other. High cohesion = the module does one well-defined job.
- **Coupling** = how strongly one module depends on the internals of another.
  Low (loose) coupling = modules interact through small, stable interfaces.

**Why:** High cohesion + loose coupling is the single best predictor of a
maintainable, changeable system. Loosely coupled modules can be understood,
tested, replaced, and reused in isolation. Tightly coupled modules must be
understood and changed together, which scales poorly.

**Practical levers:**

- Depend on abstractions (interfaces), not concretions (Dependency Inversion).
- Pass dependencies in (Dependency Injection) rather than constructing them
  internally.
- Keep public surface area small.

---

## 7. SOLID

Five object-oriented design principles (Robert C. Martin) that generalize to
most modular designs.

### 7.1 Single Responsibility Principle (SRP)

**`GEN-PRIN-07` (SHOULD)** A module should have one, and only one, reason to
change — it should be responsible to a single actor/stakeholder.

**Why:** When a class serves multiple stakeholders, a change requested by one
can break functionality relied on by another. Separating responsibilities
isolates the blast radius of change.

### 7.2 Open/Closed Principle (OCP)

**`GEN-PRIN-08` (CONSIDER)** Software entities should be open for extension but
closed for modification — add new behavior by adding new code, not by editing
existing, tested code.

**Why:** Modifying working code risks regressions. Extension points (polymorphism,
strategy objects, plugins) let you add behavior without touching the stable core.

**Cons:** Designing extension points speculatively conflicts with YAGNI. Apply
OCP at boundaries that have *demonstrated* a need to vary, not everywhere.

### 7.3 Liskov Substitution Principle (LSP)

**`GEN-PRIN-09` (MUST when using inheritance)** Subtypes must be substitutable for
their base types without altering correctness. A subclass must honor the base
class's contract: not strengthen preconditions, not weaken postconditions, not
throw unexpected exceptions.

**Why:** Violating LSP breaks polymorphism — callers written against the base
type behave incorrectly with the subtype. The classic violation is
`Square extends Rectangle`, where setting width independently of height breaks
`Rectangle`'s contract.

### 7.4 Interface Segregation Principle (ISP)

**`GEN-PRIN-10` (SHOULD)** Prefer many small, client-specific interfaces over one
large, general-purpose interface. No client should be forced to depend on methods
it does not use.

**Why:** Fat interfaces couple unrelated clients; a change to one client's needs
forces recompilation/re-testing of all implementers.

### 7.5 Dependency Inversion Principle (DIP)

**`GEN-PRIN-11` (SHOULD)** High-level policy should not depend on low-level
details. Both should depend on abstractions. Abstractions should not depend on
details; details depend on abstractions.

**Why:** Inverting the dependency direction lets business logic remain stable
while volatile details (databases, frameworks, devices) are swapped. It is the
foundation of testability (inject a fake) and of hexagonal/clean architecture.

> **SOLID caveat:** SOLID is a heuristic toolkit, not dogma. Mechanically
> applying all five everywhere produces over-abstracted, hard-to-follow code.
> Apply each principle where its specific problem actually exists.

---

## 8. Composition over Inheritance

**`GEN-PRIN-12` (SHOULD)** Prefer composing behavior from small parts (has-a)
over deep inheritance hierarchies (is-a).

**Why:** Inheritance is the tightest coupling available — a subclass depends on
its parent's implementation details, and changes to the base class ripple to all
descendants ("fragile base class"). Deep hierarchies are rigid and hard to
reason about. Composition assembles behavior at runtime from independent,
testable units.

**Pros of composition**

- Flexible: change behavior by swapping components, even at runtime.
- Avoids the diamond problem and fragile base classes.
- Each component is independently testable.

**Cons of composition**

- More wiring/boilerplate (more objects, more delegation).
- Can scatter behavior across many small types.

**When inheritance is still appropriate:** genuine, stable is-a relationships
with a well-defined substitutable contract (see LSP), e.g. framework extension
points and sealed type hierarchies (algebraic data types).

---

## 9. Law of Demeter (Principle of Least Knowledge)

**`GEN-PRIN-13` (SHOULD)** A method should only talk to its immediate
collaborators — its own fields, its parameters, and objects it creates. Avoid
"train wrecks": `a.getB().getC().doSomething()`.

**Why:** Chained access couples the caller to the internal structure of distant
objects. Any change to that structure breaks the caller. Telling an object to do
something (`order.ship()`) instead of reaching through it (`order.getWarehouse()
.getDock().load()`) keeps responsibilities and knowledge local.

**Cons:** Strict application can create many thin delegating "wrapper" methods.
Treat it as a smell detector, not an absolute law — fluent/builder APIs and pure
data structures are reasonable exceptions.

---

## 10. Encapsulation & Information Hiding

**`GEN-PRIN-14` (SHOULD)** Hide internal state and implementation details behind a
minimal public interface. Default to the most restrictive visibility; widen only
with deliberate intent.

**Why:** What is public is a contract you must preserve. The smaller the public
surface, the more freely you can change internals without breaking callers. This
is the practical mechanism that enables loose coupling and SemVer guarantees.

---

## 11. Principle of Least Astonishment

**`GEN-PRIN-15` (SHOULD)** Code should behave the way a competent reader would
reasonably expect. Names, return types, side effects, and error behavior should
not surprise.

**Why:** Surprising behavior (a "getter" that mutates state, a function that
sometimes returns `null` and sometimes throws) causes bugs because callers reason
about the expected behavior, not the actual one.

---

## 12. Naming

Naming is one of the highest-leverage readability tools.

**`GEN-PRIN-16` (SHOULD)** Use intention-revealing names: a name should answer
*why it exists, what it does, and how it is used* without a comment.

**`GEN-PRIN-17` (SHOULD)** Make names searchable and pronounceable. Avoid
single-letter names except for tight, conventional scopes (loop indices `i`,
math `x/y`).

**`GEN-PRIN-18` (SHOULD)** Use consistent vocabulary: one concept, one word
(don't mix `fetch`, `get`, `retrieve` for the same operation).

**`GEN-PRIN-19` (SHOULD)** Name length should scale with scope: short names for
short-lived locals, descriptive names for widely-visible entities.

**`GEN-PRIN-20` (SHOULD)** Avoid encodings and disinformation: no Hungarian
notation in modern typed languages; don't call something a `list` if it's a set.

| Construct | Typical role | Naming guidance |
|-----------|--------------|-----------------|
| Class / type | A thing | Noun or noun phrase (`Invoice`, `HttpClient`) |
| Function / method | An action | Verb or verb phrase (`calculateTotal`, `isValid`) |
| Boolean | A predicate | `is/has/can/should` prefix (`isReady`, `hasNext`) |
| Constant | A fixed value | Descriptive, units in name if relevant (`MAX_RETRIES`, `TIMEOUT_MS`) |
| Collection | Many things | Plural (`users`, `pendingJobs`) |

> Casing conventions (camelCase/PascalCase/snake_case/UPPER_CASE) are
> **language-specific** — see the relevant `languages/` file.

---

## 13. Function & Module Design

**`GEN-PRIN-21` (SHOULD)** Functions should be small and do one thing at a single
level of abstraction.

**Why:** Small, single-purpose functions are easier to name, test, reuse, and
reason about. Mixing levels of abstraction in one function (high-level policy
interleaved with low-level byte manipulation) forces the reader to constantly
shift mental gears.

**`GEN-PRIN-22` (SHOULD)** Minimize the number of parameters. Zero–two is ideal;
three is a warning; four or more usually signals a missing object or struct.

**Why:** Many parameters increase the combinatorial test space and the chance of
positional mistakes. Group related parameters into a parameter object.

**`GEN-PRIN-23` (SHOULD)** Avoid boolean "flag" parameters that switch behavior —
they signal a function doing more than one thing. Prefer two well-named functions.

**`GEN-PRIN-24` (SHOULD)** Prefer pure functions (output depends only on input, no
side effects) for core logic; isolate side effects (I/O, mutation, time,
randomness) at the edges.

**Why purity matters:** Pure functions are deterministic, trivially testable
(no mocks/setup), safely cacheable, and safely parallelizable. Pushing I/O to the
boundary ("functional core, imperative shell") is one of the most effective
testability and concurrency strategies available.

**`GEN-PRIN-25` (SHOULD)** Use guard clauses / early returns to keep the happy
path un-indented, instead of deep `if/else` nesting (the "arrow anti-pattern").

```
# Bad: deep nesting
def process(order):
    if order is not None:
        if order.is_valid():
            if order.items:
                ...  # real work buried 3 levels deep

# Good: guard clauses, flat happy path
def process(order):
    if order is None:
        raise ValueError("order required")
    if not order.is_valid():
        raise ValueError("invalid order")
    if not order.items:
        return EmptyResult()
    ...  # real work at top level
```

---

## 14. Immutability by Default

**`GEN-PRIN-26` (SHOULD)** Make data immutable unless mutation is required. Prefer
`const`/`final`/`readonly`/frozen values and value objects.

**Why:** Immutable data cannot be changed unexpectedly by another part of the
program, eliminating an entire class of aliasing and concurrency bugs. It makes
code easier to reason about: once you read a value, it stays valid.

**Pros**

- Thread-safe by construction (no data races on read-only data).
- Easier reasoning, safe sharing, free defensive copying.
- Enables caching/memoization and value equality.

**Cons**

- Can increase allocation/copying for large structures (mitigated by persistent
  data structures or builders).
- In hot paths or memory-constrained/embedded contexts, controlled mutation may
  be necessary for performance — measure first.

---

## 15. Fail Fast / Make Illegal States Unrepresentable

**`GEN-PRIN-27` (SHOULD)** Detect and report errors as close to their cause as
possible (fail fast). Validate inputs at boundaries; don't let bad data propagate.

**`GEN-PRIN-28` (SHOULD)** Design types so that invalid combinations cannot be
constructed. Encode invariants in the type system rather than checking them at
runtime everywhere.

**Why:** A program that crashes immediately at the point of a contract violation
is far easier to debug than one that limps along and corrupts data three modules
later. Making illegal states unrepresentable (e.g. a sum type for
`Loading | Loaded | Error` instead of a struct with nullable fields and boolean
flags) moves whole categories of bugs from runtime to compile time.

See [01-defensive-programming.md](02-defensive-programming.md) for the detailed
treatment, including trade-offs between fail-fast and graceful degradation.

---

## 16. Abstraction & Leaky Abstractions

**`GEN-PRIN-29` (CONSIDER)** Provide abstractions that hide irrelevant detail —
but be aware that **all non-trivial abstractions leak** (Joel Spolsky's Law of
Leaky Abstractions).

**Why it matters:** An abstraction reduces cognitive load only while it holds.
When it leaks (performance characteristics, error conditions, ordering), the user
must understand the layer beneath. Therefore: don't over-abstract, document the
edges where the abstraction leaks, and don't assume an abstraction frees you from
understanding what it wraps.

---

## 17. Semantic Versioning & API stability

**`GEN-PRIN-30` (SHOULD)** Version public interfaces with Semantic Versioning
(`MAJOR.MINOR.PATCH`):

- **MAJOR** — incompatible/breaking API changes.
- **MINOR** — backward-compatible new functionality.
- **PATCH** — backward-compatible bug fixes.

**Why:** SemVer is a contract with your consumers. It lets them upgrade safely
and lets automated tooling reason about compatibility. The discipline of deciding
"is this a breaking change?" also forces clarity about what your public API
actually is — which feeds directly into [encapsulation](#10-encapsulation--information-hiding).

**Practical rules:**

- Anything documented/exported is public and bound by SemVer; everything else is
  internal and changeable.
- Deprecate before removing; provide a migration path.
- `0.y.z` is the unstable phase — anything may change.

**`GEN-PRIN-32` (SHOULD)** Treat **public APIs, persisted data, wire/serialization
formats, database schemas/migrations, and established user workflows** as
compatibility contracts — not just source-level signatures. Breaking any of them
can cause data loss, outages, or downstream build failures.

- **Add before removing**; run old and new in parallel during transitions.
- **Version** external contracts (API versions, schema versions).
- Make migrations **reversible or at least restart-safe** where possible.
- Don't add backward-compatibility shims without a *concrete* compatibility need
  (`GEN-PRIN-03` YAGNI) — but once a contract is public, honor it.

---

## 18. Technical debt

**`GEN-PRIN-31` (CONSIDER)** Treat technical debt as a deliberate, tracked,
financial-style decision, not an accident.

**Why:** Some debt is rational (ship now, refactor after validating the market).
The danger is *unmanaged* debt: shortcuts that are never recorded and compound
into systemic fragility. Make debt visible (tracked issues, `TODO` with context,
ADRs) and pay it down on a schedule.

**Pros of deliberately taking on debt**

- Faster time-to-market / faster learning.
- Defers cost that may never need to be paid (if the feature is cut).

**Cons**

- Interest compounds: shortcuts slow every future change.
- Hidden debt erodes estimates and morale.

---

## Quick checklist

- [ ] Optimized for the reader, not the writer.
- [ ] Simplest design that meets *current* requirements (KISS, YAGNI).
- [ ] Each piece of knowledge has one authoritative home — but no false-DRY
      coupling of unrelated concepts (Rule of Three).
- [ ] High cohesion within modules; loose coupling between them.
- [ ] Dependencies point at abstractions and are injected, not hard-wired.
- [ ] Inheritance used only for genuine substitutable is-a; otherwise compose.
- [ ] Public surface area minimized; internals hidden.
- [ ] Names reveal intent; one concept = one word; length scales with scope.
- [ ] Functions are small, single-purpose, single-level-of-abstraction, few
      params, no behavior-flag booleans.
- [ ] Core logic is pure; side effects pushed to the edges.
- [ ] Guard clauses keep the happy path flat.
- [ ] Data is immutable unless mutation is justified.
- [ ] Invalid states are unrepresentable; inputs validated at boundaries.
- [ ] Public APIs are versioned with SemVer; breaking changes bump MAJOR.
- [ ] Compatibility contracts (APIs, persisted data, wire formats, schemas,
      workflows) preserved: add-before-remove, versioned, restart-safe migrations.
- [ ] Technical debt is tracked and scheduled, not hidden.

---

## References

- Robert C. Martin, *Clean Code* and *Agile Software Development: Principles,
  Patterns, and Practices* (SOLID).
- Andrew Hunt & David Thomas, *The Pragmatic Programmer* (DRY, orthogonality).
- Sandi Metz, "The Wrong Abstraction" — https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction
- Joel Spolsky, "The Law of Leaky Abstractions" — https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/
- Semantic Versioning — https://semver.org/
- Martin Fowler, "TechnicalDebt" — https://martinfowler.com/bliki/TechnicalDebt.html
- The Zen of Python (PEP 20) — https://peps.python.org/pep-0020/
