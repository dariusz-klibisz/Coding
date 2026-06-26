# Embedded C Review Checklist

> Use with [`../languages/embedded-c.md`](../languages/embedded-c.md) and
> [`general.md`](general.md).

- [ ] C standard / compiler version / target / warning flags known and pinned;
      MISRA/CERT static analysis in CI with documented deviations (`EC-STD-*`,
      `EC-TOOL-*`).
- [ ] `-Wall -Wextra -Werror -Wconversion`; formatter committed (`EC-TOOL-*`).
- [ ] Fixed-width types where width matters (registers/protocols/persistence);
      readable/semantic types elsewhere — not blanket `uintN_t` (`EC-TYPE-01`).
- [ ] No intentional UB/IDB unless isolated & documented; no signed overflow /
      div-by-zero / bad shifts / sequence-point UB (`EC-EXP-*`).
- [ ] No strict-aliasing violations (use `memcpy`/`union`); no unaligned access on
      strict cores (`EC-EXP-07`/`08`).
- [ ] No dynamic allocation in real-time paths (static/pool/arena); worst-case
      stack proven incl. ISR nesting; pattern-fill high-water check; no recursion
      (`EC-MEM-*`).
- [ ] `const` for flash data; `volatile` for registers & ISR-shared vars — never as
      a sync primitive (`EC-QUAL-*`).
- [ ] Macros parenthesized/side-effect-free or replaced by `static inline`/`enum`/
      const; headers guarded; declarations-only in headers (`EC-PRE-*`).
- [ ] Braces on all control statements; `switch` has `default`+`break`; loops
      bounded; `goto` only for forward cleanup (`EC-CTRL-*`).
- [ ] Return/status values checked; full prototypes; internal funcs `static`
      (`EC-FUNC-*`).
- [ ] ISRs short/deterministic; no blocking/`malloc`/`printf`; ISR-shared data
      atomic; SPSC ring buffers for hand-off (`EC-ISR-*`).
- [ ] Inter-task/core/DMA data protected (atomics/critical sections/RTOS mutexes
      with priority inheritance); barriers where needed; DMA cache coherency handled
      (`EC-CONC-*`).
- [ ] Registers via `volatile`/CMSIS; side effects/ordering/barriers handled; wire
      formats serialized explicitly (no struct-cast parsing); layout
      `_Static_assert`-ed (`EC-MMIO-*`, `EC-PACK-*`).
- [ ] Status-code errors with centralized cleanup; defined safe state; watchdog
      serviced on real progress; hard-fault handler (`EC-ERR-*`).
- [ ] Inputs/sensor data validated (range/CRC); reset cause read; inputs debounced;
      brownout handled; release-build assert behavior defined (`EC-DEF-*`).
- [ ] Host-based unit tests with mocked HAL; coverage (MC/DC for high SIL); fuzz
      protocol parsers (`EC-TEST-*`).
