# Concurrency & Parallelism

> Writing correct concurrent code: the hazards, the models, and the patterns that
> make multithreaded and asynchronous programs safe.

**Rule ID prefix:** `GEN-CONC`

---

## Table of contents

1. [Concurrency vs parallelism](#1-concurrency-vs-parallelism)
2. [The core hazards](#2-the-core-hazards)
3. [The cardinal rule: avoid shared mutable state](#3-the-cardinal-rule-avoid-shared-mutable-state)
4. [Synchronization primitives](#4-synchronization-primitives)
5. [Deadlock and how to avoid it](#5-deadlock-and-how-to-avoid-it)
6. [Memory models & visibility](#6-memory-models--visibility)
7. [Concurrency models compared](#7-concurrency-models-compared)
8. [Async/await](#8-asyncawait)
9. [Threads vs async vs processes](#9-threads-vs-async-vs-processes)
10. [Patterns](#10-patterns)
11. [Testing concurrent code](#11-testing-concurrent-code)
12. [Anti-patterns](#12-anti-patterns)
13. [Quick checklist](#quick-checklist)
14. [References](#references)

---

## 1. Concurrency vs parallelism

- **Concurrency** — *dealing with* many things at once: structuring a program as
  independent tasks that may overlap in time (even on one core).
- **Parallelism** — *doing* many things at once: literally executing on multiple
  cores simultaneously.

**Why it matters:** they solve different problems. Concurrency is mainly about
*throughput of I/O-bound work* and program structure; parallelism is about
*speeding up CPU-bound work*. The right tool depends on which you have (see §9 and
[05-performance.md](06-performance.md)).

---

## 2. The core hazards

Concurrency introduces bugs that are nondeterministic, hard to reproduce, and
hard to test — which is why the strategies below favor *avoiding* shared state
over *carefully sharing* it.

| Hazard | What happens | Cause |
|--------|--------------|-------|
| **Race condition** | Result depends on unpredictable timing/interleaving | Unsynchronized access to shared mutable state |
| **Data race** | Two threads access the same memory, ≥1 writes, no ordering | Missing synchronization; often UB |
| **Deadlock** | Threads wait on each other forever | Circular lock acquisition |
| **Livelock** | Threads keep changing state but make no progress | Mutual "politeness"/retry loops |
| **Starvation** | A thread never gets a needed resource | Unfair scheduling/locking |
| **Atomicity violation** | A "check-then-act" sequence interleaves | Non-atomic compound operations |
| **Lost update / torn read** | Updates overwritten; partially-written values read | Non-atomic read-modify-write |

**`GEN-CONC-01` (MUST)** Treat every access to shared mutable state from multiple
threads as a bug until proven synchronized. The absence of an observed failure is
not proof of correctness — races are timing-dependent.

---

## 3. The cardinal rule: avoid shared mutable state

**`GEN-CONC-02` (SHOULD)** The most reliable way to write correct concurrent code
is to **not share mutable state**. In order of preference:

1. **No sharing** — give each task its own data (partitioning, message passing).
2. **Share immutable data** — read-only data needs no synchronization
   (`GEN-PRIN-26`).
3. **Share via safe abstractions** — concurrent collections, atomics, actors,
   channels — instead of hand-rolled locking.
4. **Share mutable state with explicit locks** — last resort; easiest to get
   wrong.

**Why:** Most concurrency bugs stem from shared mutable state. Remove the
"shared" or the "mutable" and the bug class disappears. Immutability and
message-passing convert hard timing bugs into ordinary data flow.

---

## 4. Synchronization primitives

| Primitive | Use |
|-----------|-----|
| **Mutex / lock** | Mutual exclusion around a critical section |
| **Read/write lock** | Many readers or one writer (read-heavy data) |
| **Semaphore** | Bound concurrent access to N resources |
| **Atomic** | Lock-free single-variable read-modify-write (counters, flags, CAS) |
| **Condition variable** | Wait for a state change without busy-waiting |
| **Barrier / latch** | Synchronize phases across threads |
| **Concurrent collections** | Pre-built thread-safe data structures |

**`GEN-CONC-03` (SHOULD)** Hold locks for the shortest time possible and never
perform blocking I/O, callbacks, or long computation while holding a lock — it
serializes throughput and invites deadlock.

**`GEN-CONC-04` (MUST)** Guard *all* accesses (reads included) to a piece of
shared mutable state with the *same* lock. A single unguarded read can observe a
torn or stale value.

**`GEN-CONC-05` (SHOULD)** Make compound "check-then-act" / "read-modify-write"
operations atomic (single lock scope or a CAS/atomic), not two separate
synchronized steps.

```python
# Race: check and act are separate — another thread can insert between them
if key not in cache:          # check
    cache[key] = compute()    # act  -> two threads both compute & store

# Atomic alternative: use a lock around the whole compound op, or setdefault
with lock:
    if key not in cache:
        cache[key] = compute()
```

---

## 5. Deadlock and how to avoid it

Deadlock requires four conditions simultaneously (Coffman): mutual exclusion,
hold-and-wait, no preemption, circular wait. Break any one to prevent it.

**`GEN-CONC-06` (SHOULD)** Acquire multiple locks in a **consistent global
order** everywhere. Circular wait is impossible if all threads lock in the same
order.

**`GEN-CONC-07` (SHOULD)** Prefer lock-free/lock-light designs; use **timeouts**
on lock acquisition (`tryLock`) so a stuck acquisition fails fast instead of
hanging forever; minimize the number of locks held at once.

**`GEN-CONC-08` (SHOULD NOT)** Don't call into unknown/external code (callbacks,
virtual methods, events) while holding a lock — it may re-enter and acquire locks
in a conflicting order.

---

## 6. Memory models & visibility

**`GEN-CONC-09` (MUST)** Understand that without proper synchronization, a write
by one thread may **never become visible** to another, or may be **reordered** by
the compiler/CPU. Correctness requires the language's memory-model guarantees, not
just mutual exclusion.

**Why:** Modern compilers and CPUs reorder and cache memory operations for speed.
"It worked on my machine" can mean a stale cached value happened to be fresh.
Visibility is established by *happens-before* relationships created by locks,
atomics with proper memory ordering, `volatile` (Java/C# — **not** C's `volatile`,
which is for hardware, not threads), or thread start/join.

**`GEN-CONC-10` (SHOULD)** Use the language's high-level concurrency tools, which
encode the correct memory barriers for you, rather than hand-rolling lock-free
code with raw atomics — lock-free programming is extremely subtle and a frequent
source of rare, catastrophic bugs.

---

## 7. Concurrency models compared

| Model | Idea | Pros | Cons |
|-------|------|------|------|
| **Shared-memory + locks** | Threads share state, guarded by locks | Direct, flexible, fast when right | Race/deadlock-prone; hard to reason about |
| **Message passing / CSP (channels)** | Tasks communicate by sending messages, no shared state | No data races by construction; composable | Copying overhead; possible channel deadlocks |
| **Actor model** | Isolated actors with private state + mailboxes | Strong isolation; scales/distributes well | Message-ordering reasoning; overhead |
| **Async/event loop** | Single thread multiplexes I/O via cooperative tasks | No locks for I/O concurrency; high I/O throughput | One blocking call stalls all; no CPU parallelism alone |
| **Data parallelism** | Same op over partitioned data (map/reduce, SIMD, GPU) | Huge speedups for parallel-friendly work | Only fits decomposable problems |
| **STM (transactional memory)** | Optimistic transactions over memory | Composable; avoids explicit locks | Retry/contention cost; limited support |

**`GEN-CONC-11` (CONSIDER)** Prefer message-passing/actor/channel models for new
concurrent designs — "Do not communicate by sharing memory; instead, share memory
by communicating" (Go proverb). They eliminate data races by construction.

---

## 8. Async/await

**`GEN-CONC-12` (SHOULD)** Use `async`/`await` for I/O-bound concurrency: it lets
one thread handle thousands of in-flight operations by suspending tasks that are
waiting, without the memory/scheduling cost of a thread each.

**`GEN-CONC-13` (MUST NOT)** Never block on async code (sync-over-async:
`.Result`, `.Wait()`, `asyncio.run` inside a running loop) — it can deadlock the
event loop / thread pool and defeats the point. Stay async "all the way up."

**`GEN-CONC-14` (MUST NOT)** Never run long CPU-bound work directly on the async
event loop — it blocks all other tasks. Offload to a worker thread/process pool.

**`GEN-CONC-15` (SHOULD)** Propagate cancellation (cancellation tokens /
`AbortSignal` / cooperative cancellation) and put timeouts on awaited operations
(`GEN-ERR-19`).

> Language specifics: C# `ConfigureAwait(false)` in libraries, Python `asyncio`
> task management and `TaskGroup`, JS/TS microtask ordering — see the respective
> `languages/` files.

---

## 9. Threads vs async vs processes

**`GEN-CONC-16` (SHOULD)** Choose the execution model by workload:

| Workload | Best tool | Why |
|----------|-----------|-----|
| **I/O-bound** (network, disk, DB) | **Async/event loop** or thread pool | Threads mostly wait; async overlaps waiting cheaply |
| **CPU-bound**, parallelizable | **Multiple processes / true parallel threads** | Needs multiple cores; bypasses interpreter locks (e.g. Python GIL) |
| **Mixed** | Async shell + offload CPU work to a pool | Keep the loop responsive |
| **Isolation/fault-tolerance needed** | **Processes** | Crash/leak containment, no shared address space |

**Why processes for CPU-bound in some runtimes:** runtimes with a global
interpreter lock (CPython) can't run threads in true parallel for CPU work;
processes (or native extensions releasing the lock) are required.

---

## 10. Patterns

- **Thread pool / executor:** reuse a bounded set of workers; bound concurrency to
  avoid resource exhaustion. Don't create unbounded threads per request.
- **Producer–consumer:** decouple producers and consumers via a thread-safe queue;
  apply backpressure when the queue fills.
- **Fork–join / map-reduce:** split work, process in parallel, combine.
- **Future/promise:** represent an async result; compose without blocking.
- **Read-copy-update / immutable snapshot:** readers see a consistent immutable
  snapshot; writers publish a new one atomically.
- **Worker per partition:** shard data so each worker owns a disjoint slice — no
  sharing, no locks.

**`GEN-CONC-17` (SHOULD)** Always **bound** concurrency (pool sizes, queue
lengths, in-flight limits) and apply **backpressure**. Unbounded concurrency
exhausts memory/threads/connections and collapses under load.

---

## 11. Testing concurrent code

**`GEN-CONC-18` (SHOULD)** Make concurrency bugs reproducible and detectable:

- Run with **race/thread sanitizers** (TSan, Go `-race`, Helgrind) in CI.
- Use stress/soak tests with many iterations and high contention.
- Inject delays/randomized scheduling to surface interleavings.
- Keep core logic deterministic and single-threaded; confine concurrency to thin,
  well-tested boundaries (easier to reason about and test).

**Why:** Ordinary tests rarely trigger the rare interleaving that causes a race.
Sanitizers detect data races even when the test happens to pass.

---

## 12. Anti-patterns

- **Shared mutable state without synchronization** (the root cause of most bugs).
- **Inconsistent lock ordering** → deadlock.
- **Holding locks across I/O / callbacks / long work.**
- **Double-checked locking** done wrong (broken without correct memory ordering).
- **Sync-over-async** (`.Result`/`.Wait()`) → thread-pool deadlock.
- **CPU-bound work on the event loop.**
- **Unbounded threads/tasks/queues** (resource exhaustion, no backpressure).
- **`sleep()`-based "synchronization"** instead of proper signaling.
- **Hand-rolled lock-free code** without deep memory-model expertise.
- **Catching/ignoring exceptions in background tasks**, silently dropping work.

---

## Quick checklist

- [ ] Shared mutable state minimized; prefer no-sharing, then immutable, then safe
      abstractions, then locks.
- [ ] All accesses (incl. reads) to shared mutable state guarded by the same lock.
- [ ] Compound check-then-act / read-modify-write made atomic.
- [ ] Locks held briefly; no I/O/callbacks/long work under lock.
- [ ] Multiple locks acquired in one consistent global order; timeouts used.
- [ ] Memory-visibility guarantees relied on (locks/atomics), not luck.
- [ ] High-level concurrency tools / message-passing preferred over raw lock-free.
- [ ] Async used for I/O-bound; no blocking on async; no CPU work on the loop.
- [ ] Cancellation and timeouts propagated.
- [ ] Execution model (async/thread/process) matches the workload.
- [ ] Concurrency and queues are bounded; backpressure applied.
- [ ] Race/thread sanitizers and stress tests run in CI.

---

## References

- Brian Goetz et al., *Java Concurrency in Practice* (still the canonical text on
  shared-memory concurrency hazards and the memory model).
- Rob Pike, "Concurrency is not Parallelism" — https://go.dev/blog/waza-talk
- Go proverbs — https://go-proverbs.github.io/
- The Coffman conditions for deadlock.
- Leslie Lamport, "Time, Clocks, and the Ordering of Events" (happens-before).
- ThreadSanitizer — https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual
