# Embedded C — Coding Standards & State-of-the-Art Practices

> **Scope:** C (C99/C11) for resource-constrained, real-time, and safety-relevant
> embedded systems (microcontrollers, firmware, RTOS, bare-metal).
> **Primary standards:** MISRA C:2012 (+ Amendments), SEI CERT C, plus common
> embedded conventions (e.g. Barr Group). Safety standards referenced: IEC 61508,
> ISO 26262, DO-178C.
> **Relationship to general docs:** extends and specializes
> [general docs](../00-index.md). Where a general rule and an embedded rule conflict,
> the embedded rule wins for embedded C.

**Rule ID prefix:** `EC`

---

## Table of contents

1. [Why embedded C is different](#1-why-embedded-c-is-different)
2. [Standards landscape](#2-standards-landscape)
3. [Tooling & enforcement](#3-tooling--enforcement)
4. [Naming conventions](#4-naming-conventions)
5. [Formatting & layout](#5-formatting--layout)
6. [Types: fixed-width and `stdint.h`](#6-types-fixed-width-and-stdinth)
7. [`const`, `volatile`, and qualifiers](#7-const-volatile-and-qualifiers)
8. [Memory management](#8-memory-management)
9. [The preprocessor](#9-the-preprocessor)
10. [Expressions, integers & undefined behavior](#10-expressions-integers--undefined-behavior)
11. [Control flow](#11-control-flow)
12. [Functions & interfaces](#12-functions--interfaces)
13. [Pointers & arrays](#13-pointers--arrays)
14. [Interrupts (ISRs)](#14-interrupts-isrs)
15. [Concurrency on embedded](#15-concurrency-on-embedded)
16. [Memory-mapped I/O & registers](#16-memory-mapped-io--registers)
16a. [Endianness, alignment & packing](#16a-endianness-alignment--packing)
17. [Error handling](#17-error-handling)
18. [Defensive & safety patterns](#18-defensive--safety-patterns)
19. [Testing embedded code](#19-testing-embedded-code)
20. [Anti-patterns](#20-anti-patterns)
21. [Quick checklist](#quick-checklist)
22. [References](#references)

---

## 1. Why embedded C is different

Embedded development inverts several assumptions of general application
programming, which is why it warrants its own standards:

- **Severe resource limits** — kilobytes of RAM/flash; every byte and cycle
  counts.
- **No (or untrusted) OS** — often bare-metal or a small RTOS; no virtual memory,
  no process isolation, no rich runtime.
- **Real-time determinism** — correctness includes *timing*; an answer that is
  late is wrong. This forbids unbounded/nondeterministic constructs.
- **Direct hardware interaction** — registers, interrupts, DMA, peripherals.
- **High cost of failure** — firmware may be unpatchable in the field and may
  control physical actuators where a bug causes injury or damage.
- **No exceptions / limited stdlib** — C has no exceptions; many stdlib facilities
  (heap, `printf` float, recursion) are avoided.

Consequence: embedded C strongly favors **simplicity, determinism, static
allocation, and provability** over flexibility and dynamism.

---

## 2. Standards landscape

### MISRA C:2012

The dominant coding standard for safety- and security-related embedded C.
Guidelines are classified as **Mandatory** (never violate), **Required** (violate
only with a formally documented deviation), and **Advisory** (recommended).
Each is **Decidable/Undecidable** and analyzable by static tools.

**`EC-STD-01` (SHOULD)** Adopt MISRA C:2012 (with Amendments 1–4 / C:2023 where
applicable) for safety-relevant code, enforced by a certified static analyzer,
and manage exceptions through a formal **deviation** process.

**Why:** MISRA restricts C to a safer subset, eliminating constructs that are
ambiguous, implementation-defined, or commonly misused — exactly the constructs
that cause field failures in long-lived, hard-to-patch firmware.

**Pros:** removes whole classes of bugs/UB; tool-enforceable; expected/required
in automotive, medical, aerospace, industrial.
**Cons:** can feel restrictive; some rules need documented deviations for
legitimate low-level code; tooling and process overhead. The deviation mechanism
exists precisely to handle justified exceptions.

### SEI CERT C

Security-focused rules and recommendations organized by topic (PRE, DCL, EXP,
INT, FLP, ARR, STR, MEM, FIO, ENV, SIG, ERR, CON, MSC). Complementary to MISRA;
emphasizes avoiding undefined behavior and vulnerabilities.

### Others

- **Barr Group Embedded C Coding Standard** — pragmatic rules specifically aimed
  at *preventing bugs* (e.g. brace and spacing rules that stop real defects).
- **IEC 61508 / ISO 26262 / DO-178C / IEC 62304** — domain safety standards
  (industrial / automotive / aerospace / medical) that *mandate* coding standards
  and verification rigor.

---

## 3. Tooling & enforcement

**`EC-TOOL-01` (MUST)** Compile clean with maximum warnings treated as errors:
`-Wall -Wextra -Werror` (GCC/Clang) or equivalent; add `-Wconversion`,
`-Wshadow`, `-Wpointer-arith`, `-Wcast-align`, `-Wstrict-prototypes`.

**`EC-TOOL-02` (SHOULD)** Run a MISRA-capable **static analyzer** (e.g. PC-lint
Plus, Polyspace, Coverity, Cppcheck with MISRA addon, Helix QAC) in CI and gate
on it.

**`EC-TOOL-03` (SHOULD)** Specify the language standard explicitly (`-std=c99` or
`-std=c11`) and `-pedantic`; never rely on compiler extensions silently.

**`EC-TOOL-04` (SHOULD)** Use a deterministic formatter (`clang-format`) with a
committed config so style is automatic and non-negotiable.

**`EC-TOOL-05` (CONSIDER)** Where the target allows, run host-based tests under
sanitizers (ASan/UBSan) and use stack-usage analysis (`-fstack-usage`) and
static stack-depth tools to prove no stack overflow.

---

## 4. Naming conventions

Embedded teams vary, but consistency + clarity is mandatory. A common, safe set:

| Construct | Convention | Example |
|-----------|------------|---------|
| Variables, functions | `snake_case` | `read_sensor`, `byte_count` |
| Types (`typedef`) | `snake_case` with `_t` suffix | `sensor_state_t` |
| Macros, constants | `UPPER_SNAKE_CASE` | `MAX_RETRIES`, `BUFFER_SIZE` |
| `enum` members | `UPPER_SNAKE`, often prefixed | `STATE_IDLE`, `STATE_RUNNING` |
| Globals (if unavoidable) | prefix `g_` | `g_system_ticks` |
| File-scope (`static`) | prefix `s_` (common) | `s_retry_count` |
| Module public API | module prefix | `uart_init`, `uart_send` |

**`EC-NAME-01` (SHOULD)** Prefix all externally-visible identifiers of a module
with a unique module name (`uart_`, `adc_`) — C has a single global namespace, so
prefixes prevent link-time collisions.

**`EC-NAME-02` (SHOULD)** Encode units in names where applicable (`timeout_ms`,
`voltage_mv`) — wrong-unit bugs are common and dangerous in embedded.

**`EC-NAME-03` (SHOULD)** Avoid names differing only by case or easily-confused
characters; keep externally-visible names distinct within the first 31 characters
(C99 linker-significance limit).

---

## 5. Formatting & layout

**`EC-FMT-01` (MUST)** Always use braces `{ }` for the body of every `if`, `else`,
`for`, `while`, and `do` — even single statements (MISRA C:2012 Rule 15.6).

**Why:** Brace-less bodies caused the Apple "goto fail" TLS vulnerability (an
unbraced `if` with a duplicated `goto` skipped certificate validation). Mandatory
braces make the controlled block unambiguous and prevent edit-time mistakes.

```c
/* WRONG (the goto fail class of bug) */
if (err != 0)
    goto fail;
    goto fail;           /* always executes! */

/* RIGHT */
if (err != 0) {
    goto fail;
}
```

**`EC-FMT-02` (SHOULD)** One statement per line; one declaration per line; cons
indentation (commonly 4 spaces, no tabs) defined in the formatter config.

**`EC-FMT-03` (SHOULD)** Keep functions short enough to see on one screen and with
low cyclomatic complexity (many standards cap it, e.g. ≤ 10–15) — complex
functions are hard to test exhaustively, which matters for safety.

---

## 6. Types: fixed-width and `stdint.h`

**`EC-TYPE-01` (MUST)** Use the fixed-width integer types from `<stdint.h>`
(`uint8_t`, `int16_t`, `uint32_t`, …) rather than plain `int`, `short`, `long`
whose sizes are implementation-defined — **wherever exact width matters**: register
maps, hardware/protocol fields, persisted/wire formats, binary interfaces, and any
value whose range or overflow behavior depends on width.

**Why:** On embedded targets, `int` may be 16 or 32 bits; relying on a particular
width is non-portable and a frequent source of overflow/truncation bugs. Register
maps, protocols, and memory layouts demand exact widths. (MISRA Dir 4.6.)

> **Don't over-apply it.** For ordinary local arithmetic (loop counters, small
> intermediate values) plain `int`/`bool` or a domain `typedef` is often clearer
> than peppering `uint8_t`/`uint32_t` everywhere, and MISRA's essential-type model
> is about *consistent, intentional* types — not about forcing fixed-width types on
> every local. Use fixed-width types where width is part of the contract; use
> readable types (incl. semantic typedefs like `motor_rpm_t`) elsewhere.

```c
/* WRONG: width depends on the target/compiler */
unsigned int flags;
long timestamp;

/* RIGHT: explicit, portable widths */
uint32_t flags;
int64_t  timestamp_us;
```

**`EC-TYPE-02` (SHOULD)** Use `bool` from `<stdbool.h>` for boolean values, not
`int` (MISRA essential-type model). Use `stdint`'s `[u]int_fast*_t` /
`[u]int_least*_t` when you need "at least N bits, fastest" instead of exact width.

**`EC-TYPE-03` (SHOULD)** Respect the MISRA *essential type model*: don't mix
signedness in arithmetic/comparisons; make conversions explicit and intentional;
beware implicit integer promotions (`uint8_t + uint8_t` promotes to `int`).

**`EC-TYPE-04` (MUST)** Avoid floating point on MCUs without an FPU unless
required; it is slow (software emulation), non-deterministic in timing, and
precision-tricky. Prefer fixed-point / integer math. If used, never compare floats
for exact equality.

---

## 7. `const`, `volatile`, and qualifiers

**`EC-QUAL-01` (SHOULD)** Apply `const` aggressively: to read-only parameters
(especially pointer-to-const for input buffers), lookup tables (placed in flash,
saving RAM), and any value that must not change.

**Why:** `const` documents intent, lets the compiler catch accidental writes, and
on embedded specifically allows `const` data to live in flash rather than precious
RAM.

**`EC-QUAL-02` (MUST)** Use `volatile` for every variable that can change outside
the normal program flow: memory-mapped hardware registers, variables shared with
an ISR, and flags modified by another execution context.

**Why:** Without `volatile`, the optimizer may cache the value in a register and
never re-read the actual memory/register — so the program misses hardware updates
or ISR changes. This is one of the most common and baffling embedded bugs.

```c
/* Flag set by ISR, polled by main loop -> MUST be volatile */
static volatile bool g_data_ready = false;

void ADC_IRQHandler(void) { g_data_ready = true; }

void main_loop(void) {
    while (!g_data_ready) { /* without volatile, may loop forever */ }
    g_data_ready = false;
}
```

**`EC-QUAL-03` (MUST NOT)** Don't assume `volatile` provides atomicity or memory
ordering between threads/cores — it does **not** (unlike Java/C# `volatile`). For
multi-byte shared data, also guarantee atomic access (disable interrupts, use C11
`<stdatomic.h>`, or a lock). See §15.

---

## 8. Memory management

**`EC-MEM-01` (SHOULD / often MUST)** Avoid dynamic memory allocation
(`malloc`/`free`) after initialization in real-time and safety-critical embedded
code. Prefer **static** or **stack** allocation with known, bounded sizes.

**Why:** Heap allocation on embedded is hazardous: it can **fail** (out of
memory), **fragment** (small heaps fragment quickly, causing later failures),
take **nondeterministic time** (breaking real-time guarantees), and **leak**
(devices run for years). Many safety standards effectively ban or heavily restrict
it. Static allocation makes worst-case memory usage provable at build time.

**Pros of static/stack allocation**

- Memory footprint known and verifiable at compile/link time.
- No fragmentation, no allocation failure at runtime, deterministic timing.

**Cons / when dynamic is acceptable**

- Less flexible; must size buffers for the worst case (can waste RAM).
- If dynamic allocation is genuinely required, restrict it to **startup only**, or
  use deterministic schemes (fixed-size memory pools / block allocators, or an
  **arena/region allocator** that is bulk-reset at a known safe point) rather than
  general `malloc`/`free`.

**`EC-MEM-02` (MUST)** Prove the **worst-case stack depth** (no unbounded
recursion — MISRA Rule 17.2 forbids recursion; bounded recursion only with
analysis). Stack overflow silently corrupts memory. Account for **interrupt
nesting** in the worst-case calculation. As a runtime cross-check, **fill the stack
with a known pattern** (e.g. `0xA5`) at startup and measure the high-water mark in
test/field to validate the static analysis.

**`EC-MEM-03` (MUST)** Initialize all variables before use; don't rely on
implicit zeroing assumptions for non-static data. Uninitialized reads are UB and a
classic embedded bug.

**`EC-MEM-04` (SHOULD)** If pools/heap are used, set freed pointers to `NULL`
after freeing (prevents use-after-free / double-free) and never use memory after
it is freed.

---

## 9. The preprocessor

The preprocessor is powerful but unsafe (text substitution, no type checking).
MISRA restricts it heavily.

**`EC-PRE-01` (SHOULD)** Prefer `const`/`enum`/`static inline` functions over
`#define` for constants and function-like behavior. Macros lack type safety,
scoping, and debuggability.

```c
/* Avoid: untyped, unscoped, no type checking */
#define MAX(a, b) ((a) > (b) ? (a) : (b))   /* double-evaluates args! */

/* Prefer: typed, debuggable, single evaluation */
static inline int32_t max_i32(int32_t a, int32_t b) {
    return (a > b) ? a : b;
}
```

**`EC-PRE-02` (MUST)** If you must write a function-like macro, fully parenthesize
every parameter and the whole expression, and never pass expressions with side
effects (macros may evaluate arguments multiple times — `MAX(i++, j++)`).

**`EC-PRE-03` (MUST)** Guard every header against multiple inclusion (include
guards or `#pragma once`), and don't put definitions (only declarations) in
headers (MISRA Rule 8.x family / one-definition discipline).

**`EC-PRE-04` (SHOULD)** Keep conditional compilation (`#ifdef`) shallow and
centralized (a config header); deep, scattered `#ifdef` thickets are unreadable
and untestable. Ensure all `#if`/`#endif` are balanced and commented.

---

## 10. Expressions, integers & undefined behavior

Undefined behavior (UB) is the deadliest C hazard: the compiler may do *anything*,
including "optimizing away" your safety checks.

**`EC-EXP-01` (MUST)** Prevent **signed integer overflow** (UB) — check before
operations that could overflow; prefer unsigned types for wrap-around-safe
counters, but guard their comparisons/subtractions against underflow. (CERT INT30/
INT32; MISRA.)

**`EC-EXP-02` (MUST)** Never divide by zero or shift by ≥ the type width / by a
negative amount (UB). Validate divisors and shift counts.

**`EC-EXP-03` (MUST)** Avoid evaluation-order and sequence-point UB: don't read and
modify the same object twice without an intervening sequence point
(`a[i] = i++;`). Don't depend on function-argument evaluation order.

**`EC-EXP-04` (SHOULD)** Make all type conversions explicit (casts) and intended;
avoid implicit narrowing/sign-change conversions (`-Wconversion`). Beware integer
promotion turning small-type arithmetic into `int`.

**`EC-EXP-05` (SHOULD)** Parenthesize sub-expressions to make precedence explicit
(MISRA Rule 12.1) rather than relying on readers to recall C's precedence table.

**`EC-EXP-06` (MUST NOT)** Don't write code that depends on implementation-defined
or unspecified behavior (e.g. signedness of `char`, struct padding, right-shift of
negative values) without isolating and documenting it.

**`EC-EXP-07` (MUST NOT)** Don't violate **strict aliasing**: don't read memory of
one type through an incompatible pointer type (e.g. casting a `float*` to `uint32_t*`
to inspect its bits, or reinterpreting a byte buffer as a struct). This is UB and
compilers exploit it. Use `memcpy` (which the compiler optimizes away) or a `union`
for type-punning, and `-fno-strict-aliasing` only as a documented last resort.

**`EC-EXP-08` (MUST)** Don't perform **unaligned access** where the architecture
forbids it (many MCUs fault or silently misread on unaligned loads/stores). Don't
cast an arbitrary byte pointer to a wider type and dereference it; copy into a
properly aligned object instead.

---

## 11. Control flow

**`EC-CTRL-01` (MUST)** Every `switch` must have a `default` case (MISRA Rule
16.4), and every non-empty `case` must end with `break` (or an explicitly
annotated, intentional fall-through). Missing `break` is a classic bug.

**`EC-CTRL-02` (SHOULD)** Aim for **single exit** from functions where it aids
clarity, *or* use early-return guard clauses consistently — but be consistent. The
single-exit style pairs well with `goto cleanup;` resource release in C.

**`EC-CTRL-03` (CONSIDER)** `goto` is restricted by MISRA (Rule 15.1 advisory
against it) but the **forward `goto cleanup;`** idiom for centralized error/
resource cleanup is widely accepted (and used in the Linux kernel) because it
avoids deeply nested error handling. Never jump backward or into a block.

```c
int do_work(void) {
    int rc = -1;
    uint8_t *buf = pool_alloc();
    if (buf == NULL) { goto done; }

    if (init_device() != 0) { goto cleanup_buf; }
    /* ... */
    rc = 0;

cleanup_buf:
    pool_free(buf);
done:
    return rc;
}
```

**`EC-CTRL-04` (MUST)** Bound every loop. Real-time code must not contain loops
whose iteration count cannot be reasoned about; infinite loops are allowed only as
the deliberate main super-loop or RTOS task body.

---

## 12. Functions & interfaces

**`EC-FUNC-01` (MUST)** Declare full prototypes for all functions (`-Wstrict-
prototypes`); use `void` for no-parameter functions (`int f(void)`, not `int
f()`). An empty parameter list disables argument checking.

**`EC-FUNC-02` (SHOULD)** Give every function file-internal to a module `static`
linkage; expose only the intended API in the header. Minimizes the global
namespace and coupling.

**`EC-FUNC-03` (SHOULD)** Check and propagate return status; never ignore a
function's error return. Mark important returns so the compiler warns if ignored
(`__attribute__((warn_unused_result))`). (CERT ERR33-C / MISRA Dir 4.7.)

**`EC-FUNC-04` (SHOULD)** Keep parameter counts small; pass large structures by
`const` pointer, not by value (copying wastes stack and cycles).

**`EC-FUNC-05` (MUST)** Avoid recursion in real-time/safety code (MISRA Rule
17.2) — it makes worst-case stack usage unprovable.

**`EC-FUNC-06` (MUST NOT)** Don't use variadic functions and `printf`-family with
floating point on constrained targets unless the footprint/timing is acceptable;
prefer lightweight logging.

---

## 13. Pointers & arrays

**`EC-PTR-01` (MUST)** Bounds-check all array/pointer accesses driven by external
or computed indices. Out-of-bounds access is UB and the root of most memory-safety
vulnerabilities (CERT ARR; see [security](../04-security.md)).

**`EC-PTR-02` (SHOULD)** Pass array length alongside the pointer (C arrays decay to
pointers and lose size). Never compute size with `sizeof` on a pointer parameter.

**`EC-PTR-03` (SHOULD)** Limit pointer arithmetic; restrict it to within a single
array object (MISRA). Avoid more than two levels of pointer indirection (MISRA Rule
18.x) — they hurt readability and analyzability.

**`EC-PTR-04` (MUST)** Null-check pointers that can be null before dereference; set
pointers to `NULL` after free; never return a pointer to a local (stack) variable.

**`EC-PTR-05` (MUST NOT)** Don't cast away `const`/`volatile`, and avoid casting
between unrelated pointer types or pointer↔integer except for justified, isolated
hardware/register access (and even then, document and contain it).

---

## 14. Interrupts (ISRs)

ISRs are concurrency with the harshest constraints: they preempt the main flow,
must be fast, and can't block.

**`EC-ISR-01` (MUST)** Keep ISRs **short and bounded**. Do the minimum (read the
hardware, set a flag, enqueue data) and defer real work to the main loop / a task.
Long ISRs increase latency and can cause missed interrupts.

**`EC-ISR-02` (MUST)** Declare every variable shared between an ISR and other code
`volatile` (`EC-QUAL-02`).

**`EC-ISR-03` (MUST)** Make access to multi-byte/compound shared data **atomic**
with respect to the ISR — briefly disable interrupts around the access (a critical
section) or use a lock-free single-reader/single-writer ring buffer. A non-atomic
read can be preempted mid-update and observe a torn value.

```c
/* Atomic read of a 32-bit value shared with an ISR on an 8/16-bit MCU */
uint32_t read_tick_count(void) {
    uint32_t value;
    uint32_t primask = __get_PRIMASK();
    __disable_irq();                 /* enter critical section */
    value = g_system_ticks;          /* atomic w.r.t. the ISR now */
    __set_PRIMASK(primask);          /* restore prior state */
    return value;
}
```

**`EC-ISR-04` (MUST NOT)** Never call blocking, slow, non-reentrant, or
heap-allocating functions from an ISR (no `malloc`, no blocking I/O, no
`printf`). Keep ISRs reentrancy-safe.

**`EC-ISR-05` (SHOULD)** Use a lock-free single-producer/single-consumer ring
buffer to move data from an ISR to the main loop — it avoids disabling interrupts
on the hot path.

---

## 15. Concurrency on embedded

Applies the principles in [concurrency](../07-concurrency.md)
to bare-metal/RTOS realities.

**`EC-CONC-01` (MUST)** Protect data shared between tasks/cores/ISRs. Mechanisms,
from cheapest to heaviest: atomic single-word access → brief interrupt disable
(critical section) → RTOS mutex/semaphore → spinlock (multicore).

**`EC-CONC-02` (SHOULD)** Keep critical sections (interrupts disabled) as short as
possible — disabling interrupts increases worst-case interrupt latency and can
violate real-time deadlines.

**`EC-CONC-03` (SHOULD)** Use RTOS primitives correctly: prefer priority-inheritance
mutexes to avoid **priority inversion** (a classic cause of real-time failure —
famously the Mars Pathfinder bug); never call blocking RTOS APIs from an ISR (use
the `...FromISR` variants).

**`EC-CONC-04` (MUST)** Use C11 `<stdatomic.h>` or compiler/CPU memory barriers
(not bare `volatile`) when you need atomicity and ordering guarantees, especially
on multicore. `volatile` alone is insufficient for inter-thread/inter-core
synchronization (`EC-QUAL-03`).

**`EC-CONC-05` (SHOULD)** Watch for the **ABA problem** and ensure correct memory
ordering in any lock-free code; prefer well-reviewed primitives over hand-rolled
lock-free algorithms.

**`EC-CONC-06` (MUST)** Handle **DMA cache coherency** on cores with data caches:
clean (flush) the cache before a DMA read of a buffer the CPU wrote, and invalidate
the cache after a DMA write before the CPU reads it. Place DMA buffers in
non-cacheable regions or aligned to cache-line boundaries as the platform requires.
Treat DMA-shared memory as shared mutable state (`GEN-CONC-04`).

---

## 16. Memory-mapped I/O & registers

**`EC-MMIO-01` (MUST)** Access hardware registers through `volatile`-qualified
definitions so reads/writes are never optimized away or reordered by the compiler.
Use the vendor CMSIS / register-definition headers where available.

**`EC-MMIO-02` (SHOULD)** Use bit-field masks and named constants for register
fields rather than magic numbers; document the datasheet reference. Be careful:
C bit-fields have implementation-defined layout — prefer explicit
shift-and-mask for portable register access.

```c
/* Explicit, portable, documented register access */
#define UART_CR_TXEN_Pos   3U
#define UART_CR_TXEN_Msk   (1U << UART_CR_TXEN_Pos)

static inline void uart_enable_tx(volatile uint32_t *cr) {
    *cr |= UART_CR_TXEN_Msk;       /* read-modify-write the control reg */
}
```

**`EC-MMIO-03` (MUST)** Mind **read/write side effects**: some registers clear on
read, or require a specific read/write order. Don't let the compiler combine or
reorder accesses; insert memory barriers (`__DMB()`/`__DSB()`) where the hardware
requires ordering (e.g. after configuring then enabling a peripheral).

**`EC-MMIO-04` (SHOULD)** Account for endianness explicitly when (de)serializing
multi-byte protocol/register data; never assume the host byte order.

---

## 16a. Endianness, alignment & packing

**`EC-PACK-01` (MUST NOT)** Don't cast a raw byte buffer to a protocol/struct
pointer and dereference it to "parse" a wire format — it relies on the host's
endianness, struct padding, and alignment (UB, see `EC-EXP-07`/`EC-EXP-08`).
Serialize and deserialize each field **explicitly**, byte by byte, with defined
byte order.

```c
/* RIGHT: explicit, portable big-endian deserialization */
uint32_t be32_decode(const uint8_t *p) {
    return ((uint32_t)p[0] << 24) | ((uint32_t)p[1] << 16)
         | ((uint32_t)p[2] <<  8) |  (uint32_t)p[3];
}
```

**`EC-PACK-02` (SHOULD)** When a layout assumption is genuinely unavoidable (e.g. a
DMA descriptor matching a hardware-defined layout), assert it with
`_Static_assert(sizeof(desc_t) == 16, "descriptor layout")` and document the
datasheet reference rather than trusting it implicitly.

**`EC-PACK-03` (SHOULD)** Treat `#pragma pack`/`__attribute__((packed))` as a tool
for matching external layouts only; accessing packed members can generate unaligned
loads/stores. Prefer explicit serialization for portability and to avoid faults on
strict-alignment cores.

---

## 17. Error handling

C has no exceptions; embedded adds determinism constraints. See
[error handling](../03-error-handling.md).

**`EC-ERR-01` (SHOULD)** Use explicit return-status codes (an `enum` or `int`)
with a consistent convention, and **check every one** (`EC-FUNC-03`). This is the
deterministic, runtime-free model embedded requires.

**`EC-ERR-02` (SHOULD)** Centralize cleanup with the `goto cleanup;` idiom
(`EC-CTRL-03`) so resources/peripherals are released on every error path.

**`EC-ERR-03` (MUST)** Define a **safe state** for the system and transition to it
on unrecoverable faults (fail-safe): de-energize actuators, set outputs to known-
safe levels, or trigger a controlled reset. A hung or undefined state can be
dangerous (`GEN-DEF` fail-safe).

**`EC-ERR-04` (MUST)** Configure and service a **watchdog timer**: if firmware
hangs (deadlock, infinite loop, corruption), the watchdog resets the device to a
known state. Don't "pet" the watchdog blindly from an ISR — verify the main loop is
actually making progress.

**`EC-ERR-05` (SHOULD)** Handle and log hard faults (e.g. ARM HardFault handler):
capture fault registers/stack frame to aid post-mortem debugging instead of
silently locking up.

---

## 18. Defensive & safety patterns

Specializes [defensive programming](../02-defensive-programming.md).

**`EC-DEF-01` (SHOULD)** Validate inputs from outside the trust boundary (comms,
sensors, user) — range, length, checksum/CRC — before acting on them. Treat sensor
data as potentially faulty.

**`EC-DEF-02` (SHOULD)** Use **assertions** for programmer-error invariants in
debug builds; ensure release-build assertion behavior is *defined* (a custom
`assert` that enters the safe state/logs, not one that's silently compiled out
leaving no handling). Never put required logic inside `assert` (`GEN-DEF-06`).

**`EC-DEF-03` (CONSIDER)** For high-integrity systems, apply hardware/defensive
techniques: redundant storage of critical variables (e.g. value + inverted copy),
CRC-protected configuration/flash, periodic RAM/ROM self-tests, plausibility
checks on sensor readings, and EDAC/ECC where available — to detect bit-flips
(radiation/EMI) and corruption.

**`EC-DEF-04` (SHOULD)** Initialize hardware into a known-safe configuration at
startup before enabling interrupts; ensure all peripherals have defined states. On
startup, **read and act on the reset cause** (power-on, watchdog, brownout, soft
reset) so the firmware can recover or enter a safe state appropriately.

**`EC-DEF-05` (SHOULD)** **Debounce** mechanical/electrical inputs (buttons,
switches, edge-triggered sensor lines) in software or hardware; raw inputs bounce
and produce spurious transitions/interrupts.

**`EC-DEF-06` (SHOULD)** Handle **brownout**: enable the brownout detector where
available and ensure that a sagging supply drives the system to a safe state rather
than executing with marginal voltage (which corrupts RAM/flash operations).

**`EC-DEF-07` (CONSIDER)** Use a defined coding subset and tooling to keep WCET
(worst-case execution time) analyzable; avoid constructs whose timing can't be
bounded.

---

## 19. Testing embedded code

**`EC-TEST-01` (SHOULD)** Maximize **host-based unit testing**: compile
hardware-independent logic on the host (PC) and test it with a normal framework
(Unity, Ceedling, CppUTest, Google Test). Abstract hardware behind interfaces so
logic is testable without the target.

**Why:** On-target testing is slow and limited; most logic bugs can be caught far
faster on the host. The hardware abstraction needed for this also improves design
(dependency inversion, `GEN-PRIN-11`).

**`EC-TEST-02` (SHOULD)** Mock the hardware layer (register access, HAL) so tests
can simulate device responses and fault conditions.

**`EC-TEST-03` (CONSIDER)** Add on-target/hardware-in-the-loop (HIL) tests for
timing-, peripheral-, and integration-sensitive behavior that the host can't
faithfully reproduce.

**`EC-TEST-04` (SHOULD)** Measure code coverage (incl. MC/DC for the highest
safety levels, as DO-178C requires), run static analysis, and fuzz any
parser/protocol-handling code (`GEN-TEST-18`).

---

## 20. Anti-patterns

- Plain `int`/`long` for sized data instead of `stdint.h` types.
- Missing `volatile` on register/ISR-shared variables (cached-value bugs).
- Assuming `volatile` gives atomicity/ordering across threads/cores.
- Dynamic allocation (`malloc`/`free`) in real-time paths; fragmentation/leaks.
- Recursion / unbounded loops in real-time code (unprovable stack/timing).
- Brace-less control statements ("goto fail" class).
- `switch` without `default` or with missing `break`.
- Function-like macros with unparenthesized/side-effecting args.
- Ignoring function return/status values.
- Long ISRs; blocking/`malloc`/`printf` inside ISRs.
- Non-atomic access to multi-byte ISR-shared data.
- Magic numbers for registers; ignoring register read/write side effects and
  barriers.
- Casting byte buffers to structs to parse wire formats (endianness/alignment/
  padding/strict-aliasing UB); unaligned access on strict-alignment cores.
- Ignoring DMA cache coherency on cached cores.
- Unhandled reset cause; unbounced inputs; no brownout handling.
- No watchdog; no defined safe state on fault.
- Relying on implementation-defined/undefined behavior.
- Testing only on target; no host-based unit tests or HAL abstraction.

---

## Quick checklist

- [ ] MISRA C:2012 + CERT C enforced via static analyzer in CI; deviations
      documented.
- [ ] `-Wall -Wextra -Werror -Wconversion`, fixed `-std`, formatter committed.
- [ ] `stdint.h`/`stdbool.h` fixed-width types; essential-type model respected; no
      implicit narrowing.
- [ ] `const` for read-only data (flash); `volatile` for registers & ISR-shared
      vars (but not used as a sync primitive).
- [ ] No dynamic allocation in real-time paths; static/pool allocation; worst-case
      stack proven; no recursion.
- [ ] Constants/inline functions over macros; macros fully parenthesized; headers
      guarded; declarations-only in headers.
- [ ] No signed overflow / div-by-zero / bad shifts / sequence-point UB; explicit
      casts; precedence parenthesized.
- [ ] Braces on all control statements; `switch` has `default` + `break`; loops
      bounded; `goto` only for forward cleanup.
- [ ] Full prototypes (`void` params); module-internal functions `static`; return
      values checked.
- [ ] Array/pointer accesses bounds-checked; lengths passed with pointers; null
      checks; freed pointers nulled.
- [ ] ISRs short/bounded; shared data atomic & `volatile`; no blocking/`malloc` in
      ISRs; SPSC ring buffers for data hand-off.
- [ ] Inter-task/core data protected (atomics/critical sections/RTOS mutexes with
      priority inheritance); barriers where needed.
- [ ] Registers accessed via `volatile`/CMSIS; side effects, ordering/barriers, and
      endianness handled.
- [ ] Wire formats serialized/deserialized explicitly (no struct-cast parsing); no
      strict-aliasing/unaligned UB; layout assumptions `_Static_assert`-ed.
- [ ] DMA cache coherency handled (clean/invalidate or non-cacheable buffers).
- [ ] Status-code error handling with centralized cleanup; defined safe state;
      watchdog serviced on real progress; hard-fault handler.
- [ ] Reset cause read at startup; inputs debounced; brownout handled.
- [ ] Inputs/sensor data validated (range/CRC); release-build assert behavior
      defined; integrity self-tests where warranted.
- [ ] Host-based unit tests with mocked HAL; coverage (MC/DC for high SIL);
      fuzzing of protocol parsers; HIL where needed.

---

## References

- MISRA C:2012 / MISRA C:2023, *Guidelines for the Use of the C Language in
  Critical Systems* — https://www.misra.org.uk/
- SEI CERT C Coding Standard — https://wiki.sei.cmu.edu/confluence/display/c/
- Barr Group, *Embedded C Coding Standard* — https://barrgroup.com/embedded-systems/books/embedded-c-coding-standard
- ISO/IEC 9899 (C standard, C99/C11/C17).
- IEC 61508 (functional safety), ISO 26262 (automotive), DO-178C (airborne),
  IEC 62304 (medical device software).
- ARM, "CMSIS" and "Cortex-M memory barrier" application notes.
- Michael Barr, "Combining C's volatile and const," and articles on `volatile`,
  watchdogs, and priority inversion — https://embeddedgurus.com/
