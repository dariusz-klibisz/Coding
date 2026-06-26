# Python Review Checklist

> Use with [`../languages/python.md`](../languages/python.md) (full rationale and
> rule IDs) and [`general.md`](general.md).

- [ ] Local style respected; otherwise PEP 8; Black/`ruff format` + Ruff lint +
      mypy/pyright in pre-commit & CI (`PY-TOOL-*`).
- [ ] Useful type hints on public functions; modern syntax (`X | None`, built-in
      generics); `Any` avoided/contained (`PY-TYPE-*`).
- [ ] No mutable default arguments; no late-binding closure bugs (`PY-PIT-01/02`).
- [ ] Specific exceptions with narrow `try`; chaining via `raise ... from`; no bare
      `except`/swallowing (`PY-ERR-*`).
- [ ] Context managers (`with`) for resources; `encoding=` on `open` (`PY-RES-*`).
- [ ] Explicit imports at top, grouped, no wildcards; no heavy import-time work
      (`PY-IMP-*`).
- [ ] `is None`; truthiness for emptiness; `isinstance` not `type(x) ==`
      (`PY-CMP-*`).
- [ ] No unsafe `eval`/`exec`/`pickle`/`yaml.load`/`shell=True`; parameterized SQL;
      `secrets` for tokens (`PY-SEC-*`).
- [ ] **`logging` (not `print`); module logger; lazy `%`-args — no f-strings in log
      calls; no secrets logged** (`PY-LOG-*`).
- [ ] Options-heavy/boolean functions use keyword-only params (`PY-IDIOM-07`).
- [ ] Concurrency model matches workload (asyncio/threads/processes vs GIL); no
      blocking on the event loop (`PY-CONC-*`).
- [ ] PEP 257 docstrings (contract, not implementation) on public APIs
      (`PY-DOC-*`).
- [ ] Deterministic pytest tests (injected clock/RNG) covering edge cases;
      Hypothesis where invariant-rich (`PY-TEST-*`).
- [ ] venv + `pyproject.toml` + lockfile + `src/` layout (`PY-PKG-*`).
