# Go Review Checklist

> Use with [`../languages/go.md`](../languages/go.md) and [`general.md`](general.md).

- [ ] gofmt/goimports clean; `go vet` + staticcheck + `go test -race`
      pass (`GO-TOOL-*`).
- [ ] Interfaces accepted, concrete types returned; interfaces kept small
      (`GO-IDIOM-01/02`).
- [ ] Every returned `error` checked; wrapped with `%w` + context
      (`GO-ERR-01/02`).
- [ ] `panic` not used for ordinary error signaling; no panic across a
      library's public API (`GO-ERR-04`).
- [ ] Every `io.Closer` closed via `defer` right after open (`GO-RES-01`).
- [ ] Shared mutable state guarded by mutex/atomic; no unsynchronized
      concurrent map/slice access (`GO-CONC-02`).
- [ ] Every goroutine has a cancellation path (context/done channel); no
      unbounded fan-out (`GO-CONC-03/06`).
- [ ] `context.Context` threaded through I/O-bound calls (`GO-CONC-04`).
- [ ] Parameterized SQL; `html/template` for HTML; `crypto/rand` for
      security-relevant randomness (`GO-SEC-*`).
- [ ] Table-driven tests; `httptest` for handlers; `govulncheck` run
      (`GO-TEST-*`, `GO-SEC-04`).
