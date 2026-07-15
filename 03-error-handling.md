# Error Handling & Resilience

> Strategies for signaling, propagating, recovering from, and observing errors.

**Rule ID prefix:** `GEN-ERR`

---

## Table of contents

1. [Classify errors first](#1-classify-errors-first)
2. [Error signaling models](#2-error-signaling-models)
3. [Exceptions: best practices](#3-exceptions-best-practices)
4. [Error/return codes](#4-errorreturn-codes)
5. [Result / Either types](#5-result--either-types)
6. [Don't swallow errors](#6-dont-swallow-errors)
7. [Where to handle: catch at the right level](#7-where-to-handle-catch-at-the-right-level)
8. [Error context & chaining](#8-error-context--chaining)
9. [Logging errors](#9-logging-errors)
10. [Resilience patterns: retry, timeout, circuit breaker, fallback](#10-resilience-patterns)
11. [Idempotency](#11-idempotency)
12. [Cleanup & resource release](#12-cleanup--resource-release)
13. [Anti-patterns](#13-anti-patterns)
14. [Quick checklist](#quick-checklist)
15. [References](#references)

---

## 1. Classify errors first

Before choosing a mechanism, classify the error. The right handling depends on
the category.

| Category | Example | Typical response |
|----------|---------|------------------|
| **Programmer error (bug)** | null deref, index out of range, broken invariant | Fail fast; fix the code. Do not "handle." |
| **Expected/recoverable** | file not found, validation failure, network blip | Handle locally or signal to caller. |
| **Unrecoverable/environmental** | out of memory, disk full, corrupt config at startup | Fail fast, log, exit/restart cleanly. |
| **Transient** | timeout, rate limit, temporary unavailability | Retry with backoff; then degrade. |

**`GEN-ERR-01` (SHOULD)** Don't try to "recover" from programmer errors — that
masks bugs. Don't crash on expected errors — that's poor UX/availability. Match
the response to the category.

---

## 2. Error signaling models

Three dominant models. They are not mutually exclusive within a codebase, but a
given module/layer should be consistent.

| Model | Carries error as | Strengths | Weaknesses |
|-------|------------------|-----------|------------|
| **Exceptions** | Out-of-band thrown object | Clean happy path; auto-propagation; rich context/stack | Invisible control flow; easy to misuse; cost when thrown |
| **Return/error codes** | In-band return value | Explicit; no hidden control flow; predictable cost | Verbose; easy to ignore; clutters signatures |
| **Result/Either types** | Typed value (`Ok|Err`) | Explicit *and* type-checked; composable; can't ignore silently | Ceremony; needs language support to be ergonomic |

**Choosing:**

- **Exceptions** suit application code in exception-first languages (C#, Python,
  Java) for *exceptional* conditions.
- **Error codes** suit C and constrained/real-time/embedded systems where there
  is no exception runtime or where deterministic cost matters.
- **Result types** suit code that wants compile-time guarantees that every error
  is handled (Rust `Result`, functional styles, TS with a `Result<T,E>` union).

---

## 3. Exceptions: best practices

**`GEN-ERR-02` (MUST)** Throw exceptions only for *exceptional* conditions, not
for ordinary control flow.

**Why:** Using exceptions for normal flow (e.g. to break a loop, or to signal
"not found" on a hot path) is slow, obscures logic, and surprises readers.

**`GEN-ERR-03` (SHOULD)** Throw the most specific exception type available, and
catch the most specific type you can handle. Avoid catching the base
`Exception`/`Throwable` except at a top-level boundary that logs and converts.

**Why:** Broad catches swallow unrelated errors (including bugs you wanted to
surface) and prevent callers from distinguishing cases. (Confirmed by both the
Python PEP 8 and Microsoft C# guidance: catch specific types, use filters.)

**`GEN-ERR-04` (SHOULD)** Preserve the original error and stack trace when
re-throwing. Use exception chaining (`raise X from Y`, `throw new X(msg, inner)`)
rather than discarding the cause.

**`GEN-ERR-05` (SHOULD NOT)** Don't use exceptions to return data. An exception
means "I could not fulfill my contract," not "here is an alternative result."

**`GEN-ERR-06` (SHOULD)** Make exceptions informative: include what failed, the
relevant inputs (no secrets), and ideally how to fix it.

**`GEN-ERR-07` (SHOULD)** Design exception hierarchies around what the *catcher*
needs to distinguish, not around where they're thrown (per PEP 8).

**Pros of exceptions**

- The happy path stays uncluttered by error plumbing.
- Errors auto-propagate until someone can handle them; hard to "forget" entirely.
- Carry rich diagnostic context (stack traces).

**Cons of exceptions**

- Control flow is invisible at the call site — you can't tell from a call whether
  it might throw.
- Easy to over-catch or under-catch.
- Throwing/catching has runtime cost; pathological use in hot loops is slow.
- Checked-exception systems (Java) can encourage swallowing to satisfy the
  compiler.

---

## 4. Error/return codes

**`GEN-ERR-08` (MUST, where used)** Check every returned error code. The dominant
failure mode of this model is *ignored* return values.

**`GEN-ERR-09` (SHOULD)** Use a single, consistent convention (e.g. `0` = success,
negative = error; or a dedicated status enum) and enforce checking with compiler
attributes (`[[nodiscard]]`, `__attribute__((warn_unused_result))`) and static
analysis.

**Why codes dominate in C/embedded:** no exception runtime, deterministic and
visible cost, no hidden allocation or stack unwinding — essential for real-time
and MISRA-compliant code. See [languages/embedded-c.md](languages/embedded-c.md).

**Pros:** explicit, predictable cost, no hidden control flow, works without a
runtime.
**Cons:** verbose; pollutes signatures and forces out-params for the "real"
value; trivially ignored unless tooling enforces checking; error context is
limited to the code's value.

---

## 5. Result / Either types

**`GEN-ERR-10` (CONSIDER)** Model fallible operations as a value of type
`Result<T, E>` / `Either<E, T>` / `Option<T>` where the language supports it
ergonomically.

**Why:** Combines the explicitness of error codes with the safety of the type
system: the error is in the return *type*, the compiler forces you to address
both arms, and results compose (`map`, `andThen`/`flatMap`) without nesting.

```ts
// TypeScript flavor
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

function parsePort(s: string): Result<number, string> {
  const n = Number(s);
  if (!Number.isInteger(n) || n < 1 || n > 65535)
    return { ok: false, error: `invalid port: ${s}` };
  return { ok: true, value: n };
}
```

**Pros:** can't silently ignore; type-checked exhaustiveness; composable; cheap
(no stack unwinding).
**Cons:** verbose without language sugar (`?` operator, pattern matching);
interop friction with exception-based libraries; can lead to deeply nested
mapping without good combinators.

---

## 6. Don't swallow errors

**`GEN-ERR-11` (MUST NOT)** Never silently discard an error.

```
# WRONG
try:
    risky()
except Exception:
    pass          # error vanishes; bug becomes invisible

# Acceptable only with explicit, documented justification:
try:
    optional_cache_warmup()
except CacheUnavailable as e:
    logger.warning("cache warmup skipped: %s", e)   # logged + intentional
```

**Why:** A swallowed error is a future debugging nightmare — the system
misbehaves with no trace of why. If you genuinely intend to ignore an error,
prove the intent: log it, comment why, and catch the *specific* type.

**`GEN-ERR-28` (MUST NOT)** Never swallow errors on writes whose absence has
legal, audit, or compliance consequences — consent records, audit trails,
financial events. Such writes must block the operation, surface the failure, or
dead-letter for guaranteed replay. Code reading such stores must also
distinguish "empty result" from "transport failure" rather than collapsing both
into a fallback value.

**Why:** A `.catch(() => {})` on a consent or audit write converts a schema
mismatch into a silent, months-long compliance outage while the API keeps
reporting success. An ordinary swallowed error (`GEN-ERR-11`) costs debugging
time; a swallowed compliance write costs the record itself, which cannot be
reconstructed after the fact.

---

## 7. Where to handle: catch at the right level

**`GEN-ERR-12` (SHOULD)** Handle an error at the layer that has enough context to
make a correct decision — and not before.

**Why:** A low-level function usually lacks the context to decide what to do about
a failure (retry? show the user? abort the transaction?). Let errors propagate to
a layer that knows. Conversely, a top-level boundary (request handler, `main`,
event loop, task runner) MUST have a catch-all that logs and produces a safe
response, so nothing escapes uncaught.

**`GEN-ERR-13` (SHOULD)** Translate errors at architectural boundaries: convert
low-level/implementation errors (e.g. `SqlException`) into domain-meaningful
errors (e.g. `OrderNotFound`) so upper layers aren't coupled to lower-layer
details. Preserve the original as the cause (`GEN-ERR-04`).

---

## 8. Error context & chaining

**`GEN-ERR-14` (SHOULD)** As an error propagates, enrich it with context
("while loading config X", "for user Y") rather than re-wrapping it opaquely.

**Why:** The stack trace says *where*; context says *what the program was trying
to do*. Together they make root-causing fast. Avoid losing the original error
when wrapping — always chain the cause.

---

## 9. Logging errors

**`GEN-ERR-15` (SHOULD)** Log an error **once**, at the place where it is finally
handled, with full context and stack trace. Don't log-and-rethrow at every level
(produces duplicate noise).

**`GEN-ERR-16` (SHOULD)** Use structured logging (key/value fields) and
appropriate levels: `ERROR` for handled failures needing attention, `WARN` for
recovered/degraded conditions, `FATAL/CRITICAL` for imminent shutdown.

**`GEN-ERR-17` (MUST)** Never log secrets, credentials, tokens, full PII, or raw
untrusted input that could enable log injection. See [03-security.md](04-security.md).

**Why:** Logs are the primary forensic tool in production. Double-logging hides
the signal; missing context wastes incident time; leaked secrets in logs are a
common breach vector.

---

## 10. Resilience patterns

For distributed/networked systems, transient failures are normal. Combine these
patterns (most libraries: Polly for .NET, tenacity for Python, etc.).

### Retry with exponential backoff + jitter

**`GEN-ERR-18` (SHOULD)** Retry only **transient, idempotent** operations, with
exponentially increasing delays plus random jitter, and a bounded maximum.

**Why:** Immediate fixed-interval retries cause "retry storms" / thundering herds
that amplify an outage. Backoff spreads load; jitter prevents synchronized
clients from retrying in lockstep.

**Cons / cautions:** retrying non-idempotent operations can double-charge,
duplicate records, etc. (see §11). Always cap attempts and total time.

### Timeout

**`GEN-ERR-19` (MUST)** Put a timeout on every blocking/remote call. A call with
no timeout can hang forever, exhausting threads/connections.

### Circuit breaker

**`GEN-ERR-20` (CONSIDER)** After N consecutive failures, "open the circuit" and
fail fast for a cooldown period instead of hammering a failing dependency.

**Pros:** prevents cascading failure, gives the failing service room to recover,
fast failure instead of slow timeouts.
**Cons:** added complexity and state; misconfiguration can reject healthy traffic;
needs monitoring.

### Fallback / graceful degradation

**`GEN-ERR-21` (CONSIDER)** Provide a degraded alternative (cached value, default,
reduced feature) when a dependency is unavailable, where correctness allows.

**Pros:** availability during partial outages.
**Cons:** can serve stale/incorrect data; must be a *conscious* choice, not an
accidental swallow.

### Bulkhead

**`GEN-ERR-22` (CONSIDER)** Isolate resource pools (thread pools, connections) per
dependency so one failing dependency can't exhaust resources needed by others.

---

## 11. Idempotency

**`GEN-ERR-23` (SHOULD)** Design operations that may be retried to be idempotent —
applying them once or multiple times yields the same result.

**Why:** Retries, at-least-once message delivery, and user double-clicks all
cause duplicate execution. Idempotency (via idempotency keys, upserts,
conditional writes, dedup) makes these safe. Without it, retry logic
(`GEN-ERR-18`) becomes dangerous.

---

## 12. Cleanup & resource release

**`GEN-ERR-24` (MUST)** Ensure resources are released on the error path, not just
the success path — via `finally`, `using`/`with`, RAII, or single-exit cleanup.
See [01-defensive-programming.md](02-defensive-programming.md) §11.

**`GEN-ERR-25` (SHOULD)** Avoid control-flow statements (`return`/`break`) inside
`finally` blocks — they can swallow the in-flight exception (explicitly warned
against in PEP 8 and equivalent guidance).

**`GEN-ERR-26` (SHOULD)** Strive for failure atomicity: if an operation fails, the
object/system should be left in the state it had before the operation (no partial
mutation). Validate before mutating; use transactions; operate on a copy then
swap.

**`GEN-ERR-27` (MUST)** No network I/O inside an open database transaction.
Download, parse, and validate before opening the transaction; fire side effects
(notifications, messages, external API calls) only after commit. Keep
transactions short.

**Why:** A remote call mid-transaction holds locks and a connection for the full
network latency (or timeout) and couples the transaction's fate to an external
system. Worse, a side effect emitted mid-transaction becomes a lie if the
transaction rolls back — the notification was sent but the state change never
happened — and re-fires on every retry (`GEN-ERR-23`).

**`GEN-ERR-29` (SHOULD)** Treat database constraints as the authority for
uniqueness and invariants; pre-checks are advisory (useful for friendly error
messages only). A check-then-write sequence is a TOCTOU race: two concurrent
requests both pass the check, then both write. Insert/update and handle the
constraint-violation error instead (the general check-then-act hazard:
`GEN-CONC-05`). For multi-step writes spanning non-transactional systems (file
upload + DB row, external API call + local write), order the steps
least-damaging-last and add a compensating action for the step that can strand
state.

---

## 13. Anti-patterns

- **Empty catch / swallowed exception** (`GEN-ERR-11`).
- **Catch-all too low in the stack**, hiding errors from layers that could act.
- **Exceptions as control flow** for ordinary cases.
- **Log-and-rethrow at every layer** (duplicate noise).
- **Returning error codes that nobody checks** (no `[[nodiscard]]`/lint).
- **Retrying non-idempotent operations** without dedup.
- **No timeout** on remote calls.
- **Leaking secrets / raw input into logs.**
- **Returning a "default" value to hide an error** (silent corruption).
- **Throwing in destructors/finalizers/cleanup**, masking the real error.
- **`.catch(() => {})` on audit/consent/financial writes** — a silent compliance
  outage while the API reports success.
- **Network calls or notifications inside an open DB transaction** — long lock
  hold; side effects that lie on rollback and re-fire on retry.
- **Check-then-write "enforcement" of uniqueness** instead of relying on the
  database constraint (TOCTOU race).

---

## Quick checklist

- [ ] Error classified (bug vs expected vs unrecoverable vs transient) and handled
      per category.
- [ ] One consistent signaling model per layer (exceptions / codes / Result).
- [ ] Exceptions used only for exceptional conditions; specific types thrown and
      caught; cause chained on rethrow.
- [ ] Return codes (if used) are always checked, enforced by tooling.
- [ ] No swallowed errors; intentional ignores are logged + justified.
- [ ] Compliance-critical writes (audit, consent, financial) never swallowed;
      they block, surface, or dead-letter; empty result distinguished from
      transport failure.
- [ ] Errors handled at the layer with enough context; boundaries translate errors.
- [ ] Top-level catch-all logs and returns a safe response.
- [ ] Errors logged once, structured, with context, no secrets.
- [ ] Remote calls have timeouts; transient idempotent ops use bounded backoff+jitter.
- [ ] Circuit breaker / fallback / bulkhead applied where dependency failure is
      likely.
- [ ] Retryable operations are idempotent.
- [ ] Resources released on all paths; failure atomicity preserved.
- [ ] No network I/O inside open transactions; side effects fired only after
      commit.
- [ ] Uniqueness/invariants enforced by constraints (violation handled), not by
      check-then-write; non-transactional multi-step writes ordered
      least-damaging-last with compensation.

---

## References

- PEP 8, "Programming Recommendations" (exception handling) — https://peps.python.org/pep-0008/
- Microsoft, .NET Coding Conventions (exception handling) — https://learn.microsoft.com/dotnet/csharp/fundamentals/coding-style/coding-conventions
- Microsoft, "Best practices for exceptions" — https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions
- Michael Nygard, *Release It!* (circuit breaker, bulkhead, timeouts).
- Joshua Bloch, *Effective Java* (failure atomicity, exceptions).
- AWS, "Timeouts, retries and backoff with jitter" — https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/
