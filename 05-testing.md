# Testing & Quality Assurance

> How to test software effectively: what to test, at what level, with what
> techniques, and the trade-offs of each.

**Rule ID prefix:** `GEN-TEST`

---

## Table of contents

1. [Why test](#1-why-test)
2. [The test pyramid (and alternatives)](#2-the-test-pyramid-and-alternatives)
3. [Properties of a good test](#3-properties-of-a-good-test-first)
4. [Test structure: Arrange-Act-Assert](#4-test-structure-arrange-act-assert)
5. [Unit testing](#5-unit-testing)
6. [Test doubles: mocks, stubs, fakes, spies](#6-test-doubles-mocks-stubs-fakes-spies)
7. [Integration & contract testing](#7-integration--contract-testing)
8. [End-to-end testing](#8-end-to-end-testing)
9. [TDD and BDD](#9-tdd-and-bdd)
10. [Property-based & fuzz testing](#10-property-based--fuzz-testing)
11. [Coverage: use and misuse](#11-coverage-use-and-misuse)
12. [Other test types](#12-other-test-types)
13. [Testability by design](#13-testability-by-design)
14. [Anti-patterns](#14-anti-patterns)
15. [Quick checklist](#quick-checklist)
16. [References](#references)

---

## 1. Why test

Automated tests exist to:

- **Catch regressions** — verify that changes don't break existing behavior.
- **Enable change** — a good suite is what makes refactoring and rapid delivery
  *safe*; without it, every change is risky and slow.
- **Document behavior** — tests are executable specifications of intent.
- **Drive design** — code that is hard to test is usually poorly designed
  (tight coupling, hidden dependencies).

**`GEN-TEST-01` (SHOULD)** Treat tests as first-class production code: reviewed,
refactored, and kept clean. Rotting tests get disabled, and disabled tests
protect nothing.

---

## 2. The test pyramid (and alternatives)

The **test pyramid** (Mike Cohn) recommends many fast, isolated tests at the
bottom and few slow, broad tests at the top:

```
        /\        E2E / UI         few   — slow, brittle, high confidence in whole
       /  \                              system
      /----\      Integration      some  — components together (DB, services)
     /      \
    /--------\    Unit             many  — fast, isolated, pinpoint failures
```

**Why this shape:** test cost (runtime, flakiness, maintenance) rises and failure
localization worsens as you go up. Many cheap unit tests give fast, precise
feedback; a few E2E tests confirm the pieces work together. Inverting the pyramid
(the "ice-cream cone": mostly manual/E2E) yields slow, flaky, hard-to-diagnose
suites.

**Modern nuance — the "testing trophy"** (Kent C. Dodds), popular in front-end/
TS: weights **integration tests** most heavily, because for I/O-bound UI/service
code integration tests give the best confidence-per-effort, while fast static
analysis (types, lint) forms the base.

**`GEN-TEST-02` (SHOULD)** Choose the distribution that matches your system: more
unit tests for algorithm/logic-heavy code; more integration tests for glue/IO-
heavy code. The constant goal is **maximum confidence per unit of cost**.

| Level | Pros | Cons |
|-------|------|------|
| Unit | Fast, precise failures, cheap, drive design | Miss integration bugs; can over-mock and test the mock |
| Integration | Catch wiring/contract bugs; realistic | Slower; more setup; harder to localize |
| E2E | Highest confidence in real user flows | Slow, flaky, expensive to maintain, vague failures |

---

## 3. Properties of a good test (FIRST)

**`GEN-TEST-03` (SHOULD)** Aim for **FIRST** tests:

- **Fast** — milliseconds; slow suites get skipped.
- **Isolated/Independent** — no order dependence, no shared mutable state; each
  test sets up and tears down its own world.
- **Repeatable** — deterministic; same result every run, any environment. No
  reliance on real time, randomness, network, or wall-clock without control.
- **Self-validating** — pass/fail is automatic (assertions), not eyeballed.
- **Timely** — written with (or before) the code, not bolted on much later.

**`GEN-TEST-04` (SHOULD)** A test should verify **one behavior** and have a name
that states that behavior (`returns_empty_list_when_no_matches`). When it fails,
the name alone should tell you what broke.

**`GEN-TEST-05` (SHOULD)** Test **behavior, not implementation.** Assert on
observable outputs/effects, not internal calls or private state, so refactors
that preserve behavior don't break tests.

---

## 4. Test structure: Arrange-Act-Assert

**`GEN-TEST-06` (SHOULD)** Structure each test in three visually distinct phases:

```python
def test_discount_applied_for_members():
    # Arrange
    cart = Cart(items=[Item(price=100)])
    member = Customer(is_member=True)

    # Act
    total = cart.total_for(member)

    # Assert
    assert total == 90
```

**Why:** AAA (a.k.a. Given-When-Then) makes tests readable and keeps them focused
on a single action. Multiple Act/Assert cycles in one test usually means it should
be split.

**`GEN-TEST-07` (SHOULD)** Prefer one logical assertion per test (one reason to
fail). Multiple physical asserts are fine if they verify one behavior.

---

## 5. Unit testing

A **unit test** exercises a small piece of behavior (a function/class) in
isolation from slow or non-deterministic collaborators.

**`GEN-TEST-08` (SHOULD)** Cover the meaningful cases: the happy path, boundary
values (0, 1, max, empty, off-by-one), and error/edge cases. Equivalence
partitioning + boundary-value analysis catch the most bugs per test.

**`GEN-TEST-09` (SHOULD)** Keep units genuinely isolated from I/O (network, disk,
clock, DB) by injecting dependencies, so tests stay Fast and Repeatable.

---

## 6. Test doubles: mocks, stubs, fakes, spies

| Double | Purpose |
|--------|---------|
| **Dummy** | Filler passed but never used. |
| **Stub** | Returns canned responses to calls. |
| **Spy** | Stub that also records how it was called. |
| **Mock** | Pre-programmed with expectations; asserts interactions. |
| **Fake** | Working lightweight implementation (e.g. in-memory DB). |

**`GEN-TEST-10` (SHOULD)** Mock at architectural boundaries (external services,
clocks, randomness), not internal collaborators you own.

**Pros of mocking**

- Isolates the unit; tests run fast and deterministically.
- Lets you simulate hard-to-reproduce conditions (timeouts, errors).

**Cons of over-mocking**

- **Tests the mock, not reality:** heavily-mocked tests pass while the real
  integration is broken (the mock encodes your *assumption* of the dependency's
  behavior, which may be wrong).
- **Brittle:** mocks that assert call sequences break on harmless refactors.
- Mock-heavy tests can become harder to read than the code under test.

**`GEN-TEST-11` (CONSIDER)** Prefer **fakes** (e.g. in-memory repository) over
elaborate mock setups where practical — they exercise real behavior and stay
robust to refactoring. Use **contract tests** (§7) to verify the fake matches the
real dependency.

---

## 7. Integration & contract testing

**Integration tests** verify that components work together (code + real DB, two
services, code + message queue).

**`GEN-TEST-12` (SHOULD)** Run integration tests against realistic dependencies
(e.g. ephemeral containers / Testcontainers, a real local DB), not mocks, to
catch wiring, serialization, schema, and configuration bugs that unit tests miss.

**`GEN-TEST-13` (CONSIDER)** Use **consumer-driven contract tests** (e.g. Pact)
between services so each side can be tested independently while guaranteeing their
interface stays compatible — getting integration confidence without slow,
all-up E2E environments.

---

## 8. End-to-end testing

E2E tests drive the whole system as a user would (browser automation, full API
flows).

**`GEN-TEST-14` (SHOULD)** Keep E2E tests few and focused on critical user
journeys (login, checkout). They give the highest confidence but are the slowest
and flakiest.

**`GEN-TEST-15` (SHOULD)** Combat flakiness: wait on conditions (not fixed
sleeps), isolate test data, make them idempotent, and quarantine/diagnose flaky
tests immediately — a flaky suite trains the team to ignore failures.

---

## 9. TDD and BDD

### Test-Driven Development (TDD)

Red → Green → Refactor: write a failing test, write the minimum code to pass,
then refactor with the test as a safety net.

**Pros**

- Forces testable, decoupled design and small units.
- Guarantees tests exist and actually fail when behavior breaks.
- Builds a comprehensive regression suite as a by-product.
- Encourages working in small, verifiable steps.

**Cons / caveats**

- Learning curve; feels slower initially.
- Poorly applied, it can lock in implementation-coupled tests that hinder
  refactoring (mitigate with `GEN-TEST-05`).
- Less suited to exploratory/spike work where the design is unknown — spike
  first, then test-drive the real implementation.

**`GEN-TEST-16` (CONSIDER)** Use TDD especially for well-specified logic,
bug-fixes (write the failing test that reproduces the bug first), and APIs.

### Behavior-Driven Development (BDD)

Expresses tests as human-readable scenarios (Given/When/Then, e.g. Gherkin/
Cucumber) to align developers, QA, and business stakeholders.

**Pros:** shared, executable specification; readable by non-engineers; focuses on
behavior/value.
**Cons:** the natural-language layer adds maintenance overhead and indirection;
valuable mainly when non-technical stakeholders actually read the scenarios —
otherwise plain unit tests are leaner.

---

## 10. Property-based & fuzz testing

### Property-based testing

**`GEN-TEST-17` (CONSIDER)** Instead of hand-picking examples, assert *properties*
that must hold for *all* inputs, and let the framework generate hundreds of
randomized cases and shrink failures to a minimal counterexample (Hypothesis
[Python], fast-check [TS], FsCheck [.NET], QuickCheck).

```python
# Example property: round-trip encode/decode is identity
@given(st.text())
def test_roundtrip(s):
    assert decode(encode(s)) == s
```

**Why:** Example-based tests only check the cases you thought of. Property tests
explore the input space and reliably find boundary bugs (empty, huge, unicode,
negative, overflow) you'd never enumerate by hand.

**Pros:** finds edge cases automatically; tests intent (invariants), not examples;
minimal counterexamples aid debugging.
**Cons:** requires identifying good invariants (a skill); randomized failures need
seed-pinning for reproducibility; can be slower.

### Fuzz testing

**`GEN-TEST-18` (CONSIDER)** For parsers, decoders, and any code processing
untrusted bytes, run a fuzzer (libFuzzer, AFL++, `go test -fuzz`, Atheris) that
feeds malformed/random input to find crashes, hangs, and memory-safety bugs.
Especially valuable for C/C++ and security-sensitive boundaries.

---

## 11. Coverage: use and misuse

**`GEN-TEST-19` (CONSIDER)** Track code coverage to find *untested* code, but do
**not** treat a coverage percentage as a quality target.

**Why coverage is a poor goal (Goodhart's Law):** 100% line coverage can be
achieved by tests that execute code without asserting anything meaningful.
Coverage measures what code *ran*, not whether behavior was *verified*. High
coverage with weak assertions gives false confidence.

**Pros of measuring coverage:** highlights blind spots; prevents whole modules
from being untested; useful trend signal.
**Cons of targeting coverage:** incentivizes assertion-free or trivial tests;
diminishing returns near 100%; can crowd out higher-value integration/E2E
testing. **Mutation testing** (e.g. Stryker, mutmut, PIT) is a stronger measure of
test *effectiveness* — it checks whether tests actually fail when the code is
deliberately broken.

**`GEN-TEST-20` (SHOULD)** Prioritize coverage of complex, high-risk, and
high-change code over uniform targets. A pragmatic team threshold (e.g. 70–80% on
new code) is fine as a floor, not a ceiling or a goal in itself.

---

## 12. Other test types

- **Regression tests:** every bug fix ships with a test reproducing the bug, so it
  can never silently return.
- **Snapshot tests:** capture output and diff on change. Useful for UI/serialized
  output; risk: easily "approved" blindly, so review snapshot diffs carefully.
- **Performance/load tests:** verify latency/throughput SLOs under expected and
  peak load; catch performance regressions (see [05-performance.md](06-performance.md)).
- **Smoke tests:** quick "is it alive?" checks after deploy.
- **Security tests:** SAST/DAST, dependency scans (see [03-security.md](04-security.md)).
- **Accessibility & visual-regression tests** for front-end.

---

## 13. Testability by design

**`GEN-TEST-21` (SHOULD)** Design for testability — it correlates strongly with
good design generally:

- **Inject dependencies** so they can be replaced with doubles/fakes.
- **Separate pure logic from I/O** ("functional core, imperative shell") so the
  bulk of logic is testable without mocks (`GEN-PRIN-24`).
- **Avoid hidden global/static state, singletons, and direct `now()`/`random()`/
  network calls** in logic — inject a clock, RNG, and client instead.
- **Keep functions small and side-effect-light** so they're cheap to exercise.

> If something is painful to test, that pain is usually feedback about a design
> flaw, not a reason to skip the test.

---

## 14. Anti-patterns

- **Ice-cream cone:** mostly manual/E2E, few unit tests — slow and flaky.
- **Testing the mock:** so much mocking that tests pass while reality is broken.
- **Assertion-free tests:** run code but verify nothing (often coverage-chasing).
- **Brittle implementation-coupled tests** that break on every refactor.
- **Interdependent/order-dependent tests** sharing mutable state.
- **Non-deterministic tests** (real time, sleeps, randomness, network) → flakiness.
- **Logic in tests** (loops/conditionals) that can itself be buggy.
- **Slow suites** that developers stop running.
- **Disabled/ignored tests** left in the codebase indefinitely.
- **Coverage as the goal** rather than confidence.

---

## Quick checklist

- [ ] Test distribution matches the system (pyramid/trophy); not an ice-cream cone.
- [ ] Tests are FIRST (Fast, Isolated, Repeatable, Self-validating, Timely).
- [ ] Each test verifies one behavior with a descriptive name; AAA structure.
- [ ] Tests assert behavior/outputs, not private implementation.
- [ ] Happy path + boundaries + error cases covered.
- [ ] Mocking limited to owned boundaries; fakes preferred; not testing the mock.
- [ ] Integration tests run against realistic dependencies; contract tests between
      services.
- [ ] Few, stable, critical-path E2E tests; flakiness actively eliminated.
- [ ] Property-based tests for invariant-rich logic; fuzzing for parsers/untrusted
      input.
- [ ] Coverage used to find gaps, not as a target; high-risk code prioritized;
      mutation testing considered.
- [ ] Every bug fix adds a reproducing regression test.
- [ ] Code designed for testability (DI, pure core, injected clock/RNG).
- [ ] Tests treated as first-class, clean, reviewed code.

---

## References

- Mike Cohn, *Succeeding with Agile* (test pyramid).
- Martin Fowler, "TestPyramid" & "Practical Test Pyramid" — https://martinfowler.com/articles/practical-test-pyramid.html
- Kent C. Dodds, "The Testing Trophy" — https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications
- Kent Beck, *Test-Driven Development: By Example*.
- Robert C. Martin, *Clean Code* (FIRST, test cleanliness).
- Gerard Meszaros, *xUnit Test Patterns* (test doubles taxonomy).
- Hypothesis docs (property-based testing) — https://hypothesis.readthedocs.io/
- Google Testing Blog — https://testing.googleblog.com/
