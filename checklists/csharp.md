# C# / .NET Review Checklist

> Use with [`../languages/csharp.md`](../languages/csharp.md) and
> [`general.md`](general.md).

- [ ] `.editorconfig` + .NET analyzers + `TreatWarningsAsErrors`; `dotnet format`
      in CI (`CS-TOOL-*`).
- [ ] Nullable reference types enabled; intent via `T?`/`T`; guard clauses use
      `ArgumentNullException.ThrowIfNull` / `ArgumentException.ThrowIfNullOrEmpty`;
      `!` avoided/justified (`CS-NULL-*`).
- [ ] Language keywords (`string`/`int`) over runtime names; `var` only when type
      is obvious; explicit type in `foreach` (`CS-TYPE-*`, `CS-VAR-*`).
- [ ] Records for **value objects** (not identity-rich entities); `init`/`required`
      for forced construction (`CS-IMM-*`).
- [ ] Specific exceptions + filters; `throw;` (not `throw ex;`); no
      `catch(Exception)`/empty catch (`CS-ERR-*`).
- [ ] All `IDisposable`/`IAsyncDisposable` disposed by owner; correct dispose
      pattern; no finalizer reliance (`CS-DISP-*`).
- [ ] Async all the way; no `.Result`/`.Wait()`; no `async void`; cancellation +
      timeouts honored; `ConfigureAwait(false)` per repo convention in libs
      (`CS-ASYNC-*`).
- [ ] Threading: lock on private objects (not `this`/`Type`/`string`); no lock
      across `await`; `Interlocked`/concurrent collections; bounded parallelism
      (`CS-THREAD-*`).
- [ ] LINQ not multiply-enumerated; APIs return read-only/empty (never null)
      collections (`CS-COL-*`).
- [ ] Constructor DI with correct lifetimes (no captive deps, no god-constructors)
      (`CS-DI-*`).
- [ ] No `BinaryFormatter`/untrusted deserialization; parameterized SQL; CSPRNG/KDF
      for crypto (`CS-SEC-*`).
- [ ] xUnit/NUnit + AAA; integration (Testcontainers) where EF/query translation
      matters; cover success/failure/cancellation (`CS-TEST-*`).
