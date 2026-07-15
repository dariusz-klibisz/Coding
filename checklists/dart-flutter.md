# Dart & Flutter Review Checklist

> Use with [`../languages/dart-flutter.md`](../languages/dart-flutter.md) and
> [`general.md`](general.md).

- [ ] `dart format` + `dart analyze` (flutter_lints) clean (`DART-TOOL-*`).
- [ ] Null safety respected; `!` avoided outside proven invariants
      (`DART-TYPE-01`).
- [ ] Records/patterns/`sealed class` for structured data and exhaustive
      `switch` over closed alternatives (`DART-IDIOM-01/02`).
- [ ] External data modeled with typed classes, not raw
      `Map<String, dynamic>` (`DART-TYPE-03`).
- [ ] Widgets small and mostly `const`/stateless; state lifted to the
      smallest common owner (`DART-STATE-01/04`).
- [ ] All controllers/subscriptions/timers disposed; `mounted` checked after
      `await` before `setState` (`DART-STATE-03`, `DART-CONC-03`).
- [ ] No CPU-heavy work on the UI isolate (`compute()`/`Isolate.run()`
      used) (`DART-CONC-02`).
- [ ] No swallowed errors; no unobserved rejected `Future`s (`DART-ERR-02/04`).
- [ ] Secrets in secure storage, not `SharedPreferences`/hardcoded; TLS/cert
      pinning enforced (`DART-SEC-*`).
- [ ] Widget tests deterministic (fakes, `pumpAndSettle`, not real sleeps)
      (`DART-TEST-*`).
