# Rust Review Checklist

> Use with [`../languages/rust.md`](../languages/rust.md) and
> [`general.md`](general.md).

- [ ] `cargo fmt --check` + `cargo clippy -- -D warnings` clean; `cargo
      audit`/`cargo deny` run (`RS-TOOL-*`).
- [ ] No unjustified `unwrap()`/`expect()` on a reachable failure path
      (`RS-TYPE-01`).
- [ ] Exhaustive `match` over enums; `#[non_exhaustive]` where types are
      expected to grow (`RS-IDIOM-01`, `RS-TYPE-03`).
- [ ] Errors propagated with `?`; `thiserror`/`anyhow` used appropriately;
      chain preserved (`RS-ERR-*`).
- [ ] `panic!` reserved for unrecoverable programmer errors, not input-driven
      failure in a public API (`RS-ERR-03`).
- [ ] `.clone()`/`Rc<RefCell<_>>`/`Arc<Mutex<_>>` used intentionally, not to
      silence the borrow checker (`RS-RES-02/03`).
- [ ] Every `unsafe` block minimal with a `// SAFETY:` comment (`RS-SEC-01`).
- [ ] No `unsafe impl Send/Sync` without a proven safety argument
      (`RS-CONC-01`).
- [ ] Async code never blocks without `spawn_blocking`; cancellation/timeouts
      explicit (`RS-CONC-03/04`).
- [ ] Parameterized SQL; CSPRNG-backed randomness for tokens/keys
      (`RS-SEC-03`).
- [ ] Unit/integration tests present; proptest/fuzzing for parsers/invariants
      (`RS-TEST-*`).
- [ ] Embedded targets: `no_std` discipline, `embedded-hal` abstractions, no
      bare `static mut` for interrupt-shared state (`RS-EMB-*`).
