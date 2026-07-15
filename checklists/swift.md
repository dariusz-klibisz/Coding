# Swift Review Checklist

> Use with [`../languages/swift.md`](../languages/swift.md) and
> [`general.md`](general.md).

- [ ] swift-format/SwiftFormat + SwiftLint clean; Swift 6 / strict
      concurrency mode where adopted (`SWIFT-TOOL-*`).
- [ ] API-Design-Guidelines naming; call sites read as phrases
      (`SWIFT-NAME-*`).
- [ ] `struct` used unless reference semantics genuinely needed
      (`SWIFT-IDIOM-01`).
- [ ] No force-unwrap/`try!`/`try?` on runtime-reachable failure
      (`SWIFT-ERR-02/04`).
- [ ] ARC retain cycles broken (`[weak self]`/`unowned`) in closures
      (`SWIFT-RES-01/02`).
- [ ] Shared mutable state isolated in an `actor`; UI code `@MainActor`; no
      manual queue synchronization (`SWIFT-CONC-02/03`).
- [ ] Cancellation propagated, not swallowed (`SWIFT-CONC-05`).
- [ ] Secrets in Keychain, never `UserDefaults`/hardcoded; ATS enforced
      (`SWIFT-SEC-01/02`).
- [ ] External data decoded via `Codable`, not loose `[String: Any]`
      (`SWIFT-SEC-03`).
- [ ] Dependencies injected as protocols for testability; async tests await
      directly, no sleep/poll (`SWIFT-TEST-*`).
