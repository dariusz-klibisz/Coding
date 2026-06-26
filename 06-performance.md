# Performance & Efficiency

> How to reason about and improve performance without sacrificing correctness or
> maintainability. The overriding theme: **measure before you optimize.**

**Rule ID prefix:** `GEN-PERF`

---

## Table of contents

1. [The golden rule: measure first](#1-the-golden-rule-measure-first)
2. [Algorithmic complexity](#2-algorithmic-complexity)
3. [Profiling & benchmarking](#3-profiling--benchmarking)
4. [Memory & allocation](#4-memory--allocation)
5. [Data-oriented design & locality](#5-data-oriented-design--locality)
6. [I/O and the network](#6-io-and-the-network)
7. [Caching](#7-caching)
8. [Concurrency & parallelism for throughput](#8-concurrency--parallelism-for-throughput)
9. [Lazy vs eager](#9-lazy-vs-eager-evaluation)
10. [Performance and maintainability trade-off](#10-performance-vs-maintainability)
11. [Anti-patterns](#11-anti-patterns)
12. [Quick checklist](#quick-checklist)
13. [References](#references)

---

## 1. The golden rule: measure first

**`GEN-PERF-01` (MUST)** Profile before optimizing. Optimize based on measured
data, not intuition.

**Why:** Programmer intuition about *where* time/memory goes is famously wrong.
Most of a program's time is spent in a small fraction of the code; optimizing
anywhere else is wasted effort that adds complexity for no gain. This is the
substance of Knuth's full quote:

> "We should forget about small efficiencies, say about 97% of the time:
> **premature optimization is the root of all evil.** Yet we should not pass up
> our opportunities in that critical 3%." — Donald Knuth

**`GEN-PERF-02` (SHOULD)** Define the performance goal first (latency target,
throughput, memory budget, p99). Without a target you can't know when to stop, and
"faster" becomes an infinite, complexity-adding chase.

**Pros of measure-first discipline**

- Effort goes where it actually matters.
- Avoids complexity that doesn't pay off.
- Prevents "optimizations" that are actually slower on real hardware.

**Cons / caveats**

- Requires representative workloads and good tooling.
- Micro-benchmarks can mislead (warmup, JIT, caching, dead-code elimination);
  measure realistic scenarios, not synthetic toys.

---

## 2. Algorithmic complexity

**`GEN-PERF-03` (SHOULD)** Get the algorithm and data structure right first —
this dominates everything else as input size grows.

**Why:** Constant-factor micro-optimizations are irrelevant if the algorithm is
O(n²) on large n. Choosing a hash map (O(1) lookup) over a linear scan (O(n)), or
avoiding an accidental nested loop, can change runtime by orders of magnitude. No
amount of low-level tuning fixes a bad asymptotic class.

**Big-O cheat sheet (typical operations):**

| Structure | Lookup | Insert | Notes |
|-----------|--------|--------|-------|
| Array / list | O(n) search, O(1) index | O(1) amortized append | Cache-friendly, contiguous |
| Hash map/set | O(1) avg | O(1) avg | Unordered; hashing cost; memory overhead |
| Balanced tree / sorted | O(log n) | O(log n) | Ordered iteration, range queries |
| Linked list | O(n) | O(1) at known node | Poor cache locality |

**`GEN-PERF-04` (SHOULD)** Watch for hidden quadratics: nested iteration over the
same collection, repeated linear `contains` inside a loop (use a set), string
concatenation in a loop (O(n²) — use a builder/`join`), N+1 query patterns.

---

## 3. Profiling & benchmarking

**`GEN-PERF-05` (SHOULD)** Use the right tool: **CPU profilers** (sampling or
instrumenting) to find hot functions; **memory profilers/heap snapshots** for
allocation and leaks; **flame graphs** to visualize call-stack cost;
**tracing/APM** for distributed latency.

**`GEN-PERF-06` (SHOULD)** Benchmark with a proper harness that handles warmup,
multiple iterations, and statistical variance (e.g. BenchmarkDotNet, pytest-
benchmark, JMH-style). Pin the environment; beware noisy neighbors.

**`GEN-PERF-07` (SHOULD)** Optimize the **critical path** identified by the
profiler, then re-measure to confirm the change actually helped (and didn't
regress elsewhere). Optimization without re-measurement is guessing.

---

## 4. Memory & allocation

**`GEN-PERF-08` (SHOULD)** Reduce allocation in hot paths. Allocation costs CPU
directly and indirectly (GC pressure in managed languages, fragmentation in
native ones).

Techniques: reuse buffers/object pools, prefer stack/value types over heap
allocation where the language allows (`struct`, `Span<T>`, arenas), avoid
needless boxing, pre-size collections to avoid repeated reallocation, stream
instead of materializing large collections.

**`GEN-PERF-09` (SHOULD)** Understand your memory model: GC pauses (managed),
ownership/RAII (Rust/C++), manual lifetime and no heap (embedded — see
[languages/embedded-c.md](languages/embedded-c.md)). The right technique is
model-specific.

**Pros of pooling/reuse:** fewer allocations, less GC, steadier latency.
**Cons:** added complexity, risk of bugs (stale/aliased reused objects), and
premature pooling can *hurt* (defeats generational GC). Measure first.

---

## 5. Data-oriented design & locality

**`GEN-PERF-10` (CONSIDER)** For performance-critical data processing, lay out
data for cache efficiency: contiguous arrays over pointer-chasing structures,
struct-of-arrays over array-of-structs for column-wise access.

**Why:** Modern CPUs are bottlenecked on memory latency. A cache miss can cost
hundreds of cycles; sequential, contiguous access is dramatically faster than
random pointer chasing because it uses the cache and prefetcher well. This often
beats "smarter" algorithms with poor locality at realistic sizes.

**Cons:** less flexible/OO-friendly layouts; more complex code; only worth it in
measured hot paths.

---

## 6. I/O and the network

**`GEN-PERF-11` (SHOULD)** I/O usually dominates local CPU work by orders of
magnitude. Minimize round trips: batch requests, use bulk APIs, paginate, and
fix N+1 query patterns before touching CPU micro-optimizations.

**`GEN-PERF-12` (SHOULD)** Use non-blocking/async I/O (or thread pools) so that
threads aren't parked waiting on I/O — this raises throughput for I/O-bound
workloads dramatically. See [06-concurrency.md](07-concurrency.md).

**`GEN-PERF-13` (SHOULD)** Compress payloads, use efficient serialization, and
push computation close to data (DB-side filtering/aggregation) rather than pulling
everything to the client.

---

## 7. Caching

**`GEN-PERF-14` (CONSIDER)** Cache expensive, frequently-requested, and relatively
stable results (memoization, HTTP caching, CDN, application/distributed cache).

**Why:** Reusing a computed/fetched result avoids repeating expensive work and is
often the single biggest lever for latency and load reduction.

**Pros**

- Large latency and load reductions for read-heavy, repeat-access workloads.

**Cons / hazards**

- **Cache invalidation is hard** ("one of the two hard problems in CS"): stale
  data causes correctness bugs.
- Added complexity, memory cost, and a new failure mode (cache stampede on
  expiry, thundering herd).
- Cache coherence across nodes is a distributed-systems problem.

**`GEN-PERF-15` (SHOULD)** Choose an explicit invalidation strategy (TTL,
write-through, write-behind, event-based), bound cache size with an eviction
policy (LRU/LFU), and protect against stampedes (request coalescing, jittered
TTLs). Never cache user-specific/sensitive data in a shared cache without scoping.

---

## 8. Concurrency & parallelism for throughput

**`GEN-PERF-16` (CONSIDER)** Use parallelism to speed up CPU-bound work that can
be decomposed; use concurrency/async to raise throughput of I/O-bound work.

**Why distinction matters:** parallelism (multiple cores doing CPU work at once)
helps CPU-bound problems; concurrency (overlapping waiting) helps I/O-bound ones.
Misapplying them wastes effort (e.g. threading a CPU-bound task in a GIL-limited
runtime gains nothing). Amdahl's Law bounds the speedup by the serial fraction —
beyond a point, more cores stop helping.

**Cons:** concurrency introduces races, deadlocks, and nondeterminism — a large
correctness cost. Only parallelize when the measured win justifies it. See
[06-concurrency.md](07-concurrency.md) for the safety rules.

---

## 9. Lazy vs eager evaluation

**`GEN-PERF-17` (CONSIDER)** Compute values lazily (on demand, streaming,
generators) when you may not need all of them or the dataset is large; compute
eagerly when you'll need everything and want predictable, upfront cost.

**Lazy pros:** avoids unneeded work; constant memory over huge/infinite streams;
fast time-to-first-result.
**Lazy cons:** deferred cost can surprise (work happens later, at an awkward time);
harder to reason about and debug; can hold resources open; repeated traversal
recomputes.

---

## 10. Performance vs. maintainability

**`GEN-PERF-18` (SHOULD)** Default to clear, simple code. Trade readability for
speed only in measured hot paths, and when you do, **document why** with the
benchmark numbers and isolate the optimized code behind a clean interface.

**Why:** Optimized code is usually harder to read, more bug-prone, and more
expensive to maintain. Since most code is not on the hot path (`GEN-PERF-01`),
optimizing it pays nothing while costing clarity forever. Keep the 97% clean;
spend complexity only on the critical 3%.

---

## 11. Anti-patterns

- **Premature optimization:** complicating code before profiling shows a need.
- **Optimizing the wrong thing:** tuning code that isn't on the hot path.
- **Micro-optimizing past the algorithm:** polishing an O(n²) approach.
- **No re-measurement:** "optimizing" without confirming a real improvement.
- **String concatenation in loops; N+1 queries; loading whole datasets to filter
  in memory.**
- **Caching without an invalidation strategy** (stale-data bugs) or size bound
  (memory blowup).
- **Parallelizing for its own sake**, adding races for no measured gain.
- **Benchmarks without warmup/variance handling**, yielding misleading numbers.

---

## Quick checklist

- [ ] Performance goal/target defined before optimizing.
- [ ] Profiled with representative workloads; optimizing the measured hot path.
- [ ] Algorithm/data structure chosen for the right complexity class; no hidden
      quadratics or N+1.
- [ ] Allocation reduced in hot paths; memory model understood.
- [ ] Data layout cache-friendly where it's measured to matter.
- [ ] I/O round trips minimized; async used for I/O-bound work.
- [ ] Caching has explicit invalidation + size bound + stampede protection.
- [ ] Parallelism applied to CPU-bound, concurrency to I/O-bound; justified by
      measurement.
- [ ] Re-measured after each optimization to confirm the gain.
- [ ] Non-hot-path code kept simple and readable; optimizations documented with
      numbers.

---

## References

- Donald Knuth, "Structured Programming with go to Statements" (the premature-
  optimization quote).
- Brendan Gregg, *Systems Performance* and flame graphs — https://www.brendangregg.com/
- Martin Thompson et al., "Mechanical Sympathy" / data-oriented design.
- Gene Amdahl, "Amdahl's Law."
- Latency Numbers Every Programmer Should Know — https://gist.github.com/jboner/2841832
