# General Review Checklist

> A compact, cross-cutting pre-delivery / PR-review gate for any change, in any
> language. For language-specific items, also use the matching file in this folder.
> Rule IDs point to the full rationale in the root docs.

## Correctness & design

- [ ] The change directly satisfies the requested behavior; minimal and localized.
- [ ] Simplest design that meets current requirements (`GEN-PRIN-02`/`03`); no
      speculative abstraction.
- [ ] High cohesion, low coupling; dependencies point at abstractions
      (`GEN-PRIN-06`/`11`).
- [ ] Names reveal domain intent; invalid states hard to represent
      (`GEN-PRIN-16`, `GEN-PRIN-28`).
- [ ] Public-API / persisted-data / wire-format / workflow compatibility preserved
      (`GEN-PRIN-32`).

## Defensive & errors

- [ ] Untrusted input validated at trust boundaries (`GEN-DEF-02`).
- [ ] Errors handled at the level with enough context; original cause preserved;
      nothing swallowed (`GEN-ERR-11`/`12`/`14`).
- [ ] Fail-fast vs fail-safe chosen deliberately per subsystem (`GEN-DEF-07`).

## Concurrency & resources

- [ ] Shared mutable state minimized/synchronized; no data races (`GEN-CONC-01`).
- [ ] Async work awaited or intentionally fire-and-forget with error handling;
      cancellation/timeouts propagated (`GEN-CONC-13`/`15`).
- [ ] Resource ownership/lifetime explicit; released on every path
      (`GEN-DEF-15`).

## Security

- [ ] No injection (parameterized queries, arg-vector commands); output encoded
      (`GEN-SEC-03`/`04`/`08`).
- [ ] No secrets in code/logs; authorization enforced server-side; deny by default
      (`GEN-SEC-14`/`24`/`39`).
- [ ] Resource/DoS limits where untrusted input sizes flow in (`GEN-SEC-31a`).

## Tests, observability & tooling

- [ ] Tests cover changed behavior (happy + boundary + error); deterministic; not
      just coverage-chasing (`GEN-TEST-03`/`08`).
- [ ] Diagnosable: meaningful structured logs at boundaries; no secrets logged
      (`GEN-OBS-03`/`05`/`09`).
- [ ] Formatter / linter / type-checker run; no new warnings; unrelated files
      untouched (`GEN-TOOL` / `GEN-VCS-22`).
- [ ] Comments explain *why* for non-obvious decisions only (`GEN-DOC-03`).
