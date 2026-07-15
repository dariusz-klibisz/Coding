# Kotlin Review Checklist

> Use with [`../languages/kotlin.md`](../languages/kotlin.md) and
> [`general.md`](general.md).

- [ ] ktlint + detekt clean (`KT-TOOL-*`).
- [ ] `data class` for value types; `sealed class`/`interface` + exhaustive
      `when` for closed alternatives (`KT-IDIOM-01/02`).
- [ ] `val`/immutable collections preferred; `!!` avoided outside proven
      invariants (`KT-IDIOM-03`, `KT-TYPE-01`).
- [ ] Coroutines scoped to a real lifecycle, never `GlobalScope`
      (`KT-CONC-01`).
- [ ] Dispatcher matches workload; no blocking call on `Main` without
      `withContext` (`KT-CONC-02/03`).
- [ ] `CancellationException` re-thrown, not swallowed (`KT-ERR-03`).
- [ ] One-shot events use `SharedFlow`/`Channel`, not `StateFlow`
      (`KT-CONC-04`).
- [ ] Resources closed via `use { }`; no UI-component leak into a
      longer-lived scope (`KT-RES-*`).
- [ ] Secrets in Keystore/`EncryptedSharedPreferences`; typed deserialization
      for external data (`KT-SEC-*`).
- [ ] Coroutine tests use `runTest`/`TestDispatcher`; dependencies injected
      for fakeability (`KT-TEST-*`).
