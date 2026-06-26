# Python — Coding Standards & State-of-the-Art Practices

> **Scope:** Modern Python 3 (3.10+). Application, library, scripting, and data
> code.
> **Primary sources:** PEP 8 (style), PEP 257 (docstrings), PEP 20 (Zen),
> PEP 484/526/604 (typing), PEP 621 (packaging metadata).
> **Relationship to general docs:** extends [general docs](../00-README.md). Where a
> general rule and a Python rule conflict, the Python rule wins for Python.

**Rule ID prefix:** `PY`

---

## Table of contents

1. [The Zen of Python](#1-the-zen-of-python)
2. [Tooling & enforcement](#2-tooling--enforcement)
3. [Naming conventions](#3-naming-conventions)
4. [Layout & formatting (PEP 8)](#4-layout--formatting-pep-8)
5. [Imports](#5-imports)
6. [Idioms & Pythonic style](#6-idioms--pythonic-style)
7. [EAFP vs LBYL](#7-eafp-vs-lbyl)
8. [Type hints](#8-type-hints)
9. [Data structures: dataclasses & friends](#9-data-structures-dataclasses--friends)
10. [Comparisons, truthiness, None](#10-comparisons-truthiness-none)
11. [Exceptions & error handling](#11-exceptions--error-handling)
12. [Resource management & context managers](#12-resource-management--context-managers)
13. [Docstrings](#13-docstrings)
14. [Common pitfalls](#14-common-pitfalls)
15. [Concurrency & the GIL](#15-concurrency--the-gil)
15a. [Logging](#15a-logging)
16. [Packaging & environments](#16-packaging--environments)
17. [Security](#17-security)
18. [Testing](#18-testing)
19. [Anti-patterns](#19-anti-patterns)
20. [Quick checklist](#quick-checklist)
21. [References](#references)

---

## 1. The Zen of Python

PEP 20 is the philosophical backbone. The lines that most directly guide code:

> Beautiful is better than ugly. Explicit is better than implicit. Simple is
> better than complex. Flat is better than nested. Readability counts. Errors
> should never pass silently (unless explicitly silenced). There should be one—
> and preferably only one—obvious way to do it.

**`PY-ZEN-01` (SHOULD)** When choosing between approaches, prefer the explicit,
simple, flat, readable one. "Pythonic" means leveraging the language's idioms for
clarity — not writing the cleverest possible code.

---

## 2. Tooling & enforcement

**`PY-TOOL-01` (MUST)** Use an autoformatter — **Black** (or `ruff format`) — with
the default config. Deterministic formatting ends style debate and keeps diffs
clean.

**`PY-TOOL-02` (MUST)** Use a linter — **Ruff** (fast, consolidates Flake8/isort/
pyupgrade/many plugins) — in CI and as a pre-commit hook. Ruff also sorts imports.

**`PY-TOOL-03` (SHOULD)** Run a **static type checker** — **mypy** or **pyright/
Pylance** — in strict mode on new code; gate CI on it (see §8).

**`PY-TOOL-04` (SHOULD)** Manage everything via **`pyproject.toml`** (PEP 621) and
a modern tool (uv/Poetry/Hatch/PDM). Use **pre-commit** to run format+lint+type
checks before each commit.

---

## 3. Naming conventions

Per PEP 8:

| Construct | Convention | Example |
|-----------|------------|---------|
| Module, package | `lower_snake_case` (packages: short, no underscores) | `http_client`, `utils` |
| Class, Exception, Type var | `CapWords` (`PascalCase`) | `OrderService`, `ConfigError`, `T` |
| Function, method, variable | `lower_snake_case` | `read_file`, `item_count` |
| Constant | `UPPER_SNAKE_CASE` | `MAX_RETRIES` |
| "Internal" (non-public) | leading underscore | `_helper`, `_cache` |
| Name-mangled (avoid clashes) | two leading underscores | `__private` |
| Instance method first arg | `self` | — |
| Class method first arg | `cls` | — |

**`PY-NAME-01` (SHOULD)** Never use `l`, `O`, or `I` as single-character names
(indistinguishable from `1`/`0` in many fonts — PEP 8).

**`PY-NAME-02` (SHOULD)** Suffix exception classes with `Error` when they denote
an error (`ValidationError`). Acronyms in CapWords stay all-caps (`HTTPServer`).

**`PY-NAME-03` (SHOULD)** Append a single trailing underscore to avoid keyword
clashes (`class_`, `type_`) rather than misspelling (PEP 8).

**`PY-NAME-04` (SHOULD)** Declare a module's public API with `__all__`; prefix
everything else with `_` to mark it internal (PEP 8).

---

## 4. Layout & formatting (PEP 8)

**`PY-FMT-01` (MUST)** 4 spaces per indent level. Never mix tabs and spaces
(Python forbids it). Spaces only.

**`PY-FMT-02` (SHOULD)** Line length: PEP 8 says 79 (72 for docstrings/comments);
many teams use 88 (Black default) or up to 99. Pick one in config and let the
formatter enforce it.

**`PY-FMT-03` (SHOULD)** Two blank lines around top-level functions/classes; one
blank line between methods (PEP 8).

**`PY-FMT-04` (SHOULD)** Surround binary operators with single spaces; no space
inside brackets/parentheses or before a call/index paren; no spaces around `=` for
keyword args/defaults **without** annotations, but **with** spaces when annotated
(PEP 8):

```python
def munge(input: str, sep: str = ",", limit=1000) -> list[str]: ...
#         ^ space after colon         ^ no spaces (unannotated default)
#                          ^ spaces around = (annotated default)
```

**`PY-FMT-05` (SHOULD)** Wrap long lines inside parentheses (implicit
continuation), not with backslashes; break **before** binary operators (PEP 8,
Knuth style). Use trailing commas in multi-line collections (cleaner diffs).

---

## 5. Imports

**`PY-IMP-01` (SHOULD)** One import per line; imports at the top of the file, after
the module docstring (PEP 8). Group in three blocks separated by blank lines:
1) standard library, 2) third-party, 3) local — let Ruff/isort enforce ordering.

**`PY-IMP-02` (SHOULD)** Prefer **absolute imports** (`from mypkg.sub import x`);
explicit relative imports (`from . import x`) are acceptable for intra-package
references (PEP 8).

**`PY-IMP-03` (MUST NOT)** Avoid wildcard imports (`from m import *`) — they
pollute the namespace and hide what's defined (PEP 8). One defensible exception:
re-exporting an internal API.

---

## 6. Idioms & Pythonic style

**`PY-IDIOM-01` (SHOULD)** Iterate directly over iterables; use `enumerate` for
indices and `zip` for parallel iteration — not `range(len(...))`.

```python
# Un-Pythonic
for i in range(len(items)):
    print(i, items[i])

# Pythonic
for i, item in enumerate(items):
    print(i, item)
```

**`PY-IDIOM-02` (SHOULD)** Use comprehensions (list/dict/set/generator) for simple
transformations/filters — they're clear and fast. But **don't** nest them deeply or
add side effects; fall back to a loop when a comprehension stops being readable.

**`PY-IDIOM-03` (SHOULD)** Use generators / `yield` for large or streaming
sequences to keep memory bounded (`GEN-PERF-17`).

**`PY-IDIOM-04` (SHOULD)** Unpack tuples, use multiple assignment, and prefer the
standard library: `collections` (`defaultdict`, `Counter`, `deque`), `itertools`,
`functools`, `pathlib` (over `os.path`), `enum` for enumerations.

**`PY-IDIOM-05` (SHOULD)** Use f-strings for formatting (`f"{name}: {value:.2f}"`)
— fast and readable. Avoid `%` and `str.format` for new code.

> **Exception — logging:** Do **not** use f-strings (or eager `%`/`.format`) in
> `logging` calls. Pass the format string and arguments separately so the
> interpolation is deferred and skipped when the level is disabled:
> `logger.info("processed %s in %d ms", item_id, elapsed)` — **not**
> `logger.info(f"processed {item_id} in {elapsed} ms")`. Eager f-strings in log
> calls waste work on suppressed levels and are flagged by linters
> (`logging-fstring-interpolation`). See [§15a Logging](#15a-logging).

**`PY-IDIOM-06` (SHOULD)** Define functions with `def`, not by assigning a `lambda`
to a name (PEP 8) — `def` gives a real name in tracebacks. Reserve `lambda` for
small inline callables.

**`PY-IDIOM-07` (SHOULD)** For options-heavy or boolean-bearing functions, make
options **keyword-only** with a bare `*` separator
(`def connect(host, *, timeout=30, retries=3)`). This prevents positional-argument
mistakes, makes call sites self-documenting, and lets you add/reorder options
without breaking callers (ties to `GEN-PRIN-22`/`GEN-PRIN-23`).

---

## 7. EAFP vs LBYL

Two idioms for handling possible failure:

- **EAFP** — "Easier to Ask Forgiveness than Permission": try the operation, catch
  the exception. *The Pythonic default.*
- **LBYL** — "Look Before You Leap": check preconditions first with `if`.

```python
# EAFP (preferred in Python)
try:
    value = config["timeout"]
except KeyError:
    value = DEFAULT_TIMEOUT

# LBYL
if "timeout" in config:
    value = config["timeout"]
else:
    value = DEFAULT_TIMEOUT
```

**`PY-EAFP-01` (SHOULD)** Prefer **EAFP** in Python.

**Why / Pros of EAFP:** avoids a **TOCTOU race** (the state can change between the
LBYL check and the use — e.g. a file deleted after `os.path.exists` but before
`open`); often faster on the success path (no redundant check); reads cleanly.
**Cons of EAFP:** can hide bugs if the `try` block is too broad (keep it minimal,
catch the specific exception — `GEN-ERR-03`); exceptions are costly if failure is
the *common* case, where LBYL (or `dict.get`) is better.

**`PY-EAFP-02` (SHOULD)** Use the targeted forms where they exist: `dict.get(key,
default)`, `getattr(obj, name, default)`, `dict.setdefault` — clearer than either
full idiom.

---

## 8. Type hints

**`PY-TYPE-01` (SHOULD)** Add type hints (PEP 484) to all public functions, method
signatures, and module-level variables where non-obvious; check with mypy/pyright.

**Why:** Type hints catch a large class of bugs before runtime, document intent
precisely, power IDE autocompletion/refactoring, and make large codebases
tractable. They're optional at runtime (no performance cost) but invaluable for
maintainability — the de-facto standard for professional Python.

**`PY-TYPE-02` (SHOULD)** Use modern typing syntax (3.10+): built-in generics
(`list[int]`, `dict[str, int]`), `X | Y` and `X | None` unions (PEP 604) instead
of `Optional[X]`/`Union`. Use `typing` constructs where needed: `Protocol`
(structural typing/duck-typing with checks), `Literal`, `TypedDict`, `Final`,
`Self`, `TypeAlias`, generics with `TypeVar`/PEP 695 `type` syntax.

```python
def fetch(url: str, *, retries: int = 3) -> bytes | None: ...

class Comparable(Protocol):           # structural typing
    def __lt__(self, other: object) -> bool: ...
```

**`PY-TYPE-03` (SHOULD)** Avoid `Any` — it disables checking. Prefer `object` +
narrowing, generics, or `Protocol`. Don't annotate with overly-broad types.

**`PY-TYPE-04` (SHOULD)** Annotate spaces correctly (PEP 8): `name: int = 0` (space
after colon, spaces around `=`); no space before the colon.

**Pros of typing:** earlier bugs, self-documentation, tooling.
**Cons:** verbosity, occasional fighting with the checker on dynamic patterns,
maintenance of stubs for untyped deps — mitigated by `# type: ignore[code]`
sparingly and typeshed/`types-*` stub packages.

---

## 9. Data structures: dataclasses & friends

**`PY-DATA-01` (SHOULD)** Use `@dataclass` for plain data-holding classes — it
generates `__init__`, `__repr__`, `__eq__`, etc. Use `frozen=True` for immutable
value objects (`GEN-PRIN-26`), and `slots=True` for memory/attribute-safety.

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class Point:
    x: float
    y: float
```

**`PY-DATA-02` (SHOULD)** Use `enum.Enum`/`IntEnum`/`StrEnum` for fixed sets of
constants (type-safe, self-documenting) instead of bare string/int constants.

**`PY-DATA-03` (CONSIDER)** Use **Pydantic** (or `attrs`) when you need runtime
validation/parsing of external data ("parse, don't validate", `GEN-DEF`); use
`NamedTuple`/`TypedDict` for lightweight typed records.

**`PY-DATA-04` (MUST)** Never use a **mutable default argument** — see §14.

---

## 10. Comparisons, truthiness, None

**`PY-CMP-01` (MUST)** Compare to `None` with `is`/`is not`, never `==` (PEP 8).
`None` is a singleton; `==` can be overridden and misbehave.

**`PY-CMP-02` (SHOULD)** Use truthiness for emptiness: `if not seq:` not `if
len(seq) == 0:`. But beware: `0`, `0.0`, `""`, `[]`, `{}`, `None` are all falsy —
when you mean "was a value provided?", test `if x is not None:` explicitly (PEP 8).

**`PY-CMP-03` (SHOULD)** Don't compare booleans to `True`/`False` (`if flag:`, not
`if flag == True:`); use `isinstance(x, T)` for type checks, not `type(x) == T`
(PEP 8).

**`PY-CMP-04` (SHOULD)** Use `is not` (`if x is not None`), not `not x is None`
(PEP 8). Use `startswith`/`endswith` over slicing for prefix/suffix checks.

---

## 11. Exceptions & error handling

Builds on [error handling](../03-error-handling.md).

**`PY-ERR-01` (MUST NOT)** Never use a bare `except:` (catches `SystemExit`/
`KeyboardInterrupt`); catch the **specific** exception, or `except Exception:` only
at a logging boundary (PEP 8).

**`PY-ERR-02` (MUST)** Keep the `try` block minimal — only the code that can raise
the expected exception — so you don't accidentally catch unrelated errors (PEP 8).
Use `else:` for the success path.

```python
# Good: narrow try, else for success path (PEP 8)
try:
    value = collection[key]
except KeyError:
    return key_not_found(key)
else:
    return handle_value(value)
```

**`PY-ERR-03` (SHOULD)** Derive custom exceptions from `Exception` (not
`BaseException`); design the hierarchy around what callers need to catch; suffix
with `Error` (PEP 8).

**`PY-ERR-04` (SHOULD)** Chain exceptions: `raise NewError(...) from original` to
preserve the cause; use `from None` to deliberately suppress, transferring needed
detail (PEP 8).

**`PY-ERR-05` (SHOULD NOT)** Don't `return`/`break`/`continue` from a `finally`
block — it silently cancels in-flight exceptions (PEP 8).

**`PY-ERR-06` (CONSIDER)** Use `contextlib.suppress(SpecificError)` to express
intentional ignoring clearly, rather than a `try/except/pass`.

---

## 12. Resource management & context managers

**`PY-RES-01` (MUST)** Use `with` for resources (files, locks, connections,
sessions) so they're released on every path (`GEN-DEF-15`):

```python
with open(path, encoding="utf-8") as f:   # always closed, even on error
    data = f.read()
```

**`PY-RES-02` (SHOULD)** Always pass `encoding=` to `open()` (and to anything
text-decoding) — relying on the platform default causes cross-platform bugs.

**`PY-RES-03` (SHOULD)** Write your own context managers with `@contextlib.context
manager` or `__enter__/__exit__` for setup/teardown pairs; use
`contextlib.ExitStack` for a dynamic number of resources.

---

## 13. Docstrings

**`PY-DOC-01` (SHOULD)** Write docstrings (PEP 257) for all public modules,
classes, functions, and methods. One-line docstrings keep the closing `"""` on the
same line; multi-line ones put `"""` on its own line.

**`PY-DOC-02` (SHOULD)** Use a consistent style (Google, NumPy, or reST) describing
Args, Returns, Raises — and let Sphinx/mkdocstrings render them (`GEN-DOC-07`).
Document the **contract**, not the implementation.

```python
def divide(a: float, b: float) -> float:
    """Divide ``a`` by ``b``.

    Args:
        a: Dividend.
        b: Divisor; must be non-zero.

    Returns:
        The quotient ``a / b``.

    Raises:
        ZeroDivisionError: If ``b`` is zero.
    """
```

---

## 14. Common pitfalls

**`PY-PIT-01` (MUST)** **Mutable default arguments** are evaluated **once** at
definition time and shared across calls — a classic bug. Use `None` as the
sentinel.

```python
# WRONG: the list is shared across all calls
def append(item, target=[]):
    target.append(item)
    return target

# RIGHT
def append(item, target: list | None = None):
    if target is None:
        target = []
    target.append(item)
    return target
```

**`PY-PIT-02` (SHOULD)** **Late binding in closures/loops:** a lambda/closure
capturing a loop variable sees its *final* value. Bind via a default arg
(`lambda x=x: ...`) or `functools.partial`.

**`PY-PIT-03` (SHOULD)** Don't rely on CPython's in-place `str +=` optimization;
build large strings with `"".join(parts)` (PEP 8, `GEN-PERF-04`).

**`PY-PIT-04` (SHOULD)** Be careful copying: assignment aliases; `list(x)`/`x.copy()`
is shallow; use `copy.deepcopy` for nested structures when needed.

**`PY-PIT-05` (SHOULD)** Floating-point equality is unreliable; use `math.isclose`.
For money/precision use `decimal.Decimal`.

**`PY-PIT-06` (SHOULD)** Don't mutate a list/dict while iterating it; iterate a copy
or build a new collection.

---

## 15. Concurrency & the GIL

See [concurrency](../07-concurrency.md).

**`PY-CONC-01` (SHOULD)** Choose the model by workload (the GIL serializes Python
bytecode in CPython, so threads don't parallelize pure-Python CPU work):

| Workload | Tool |
|----------|------|
| **I/O-bound, high concurrency** | `asyncio` (`async`/`await`) |
| **I/O-bound, simpler/blocking libs** | `threading` / `concurrent.futures.ThreadPoolExecutor` |
| **CPU-bound** | `multiprocessing` / `ProcessPoolExecutor` (or native extensions releasing the GIL) |

> Note: Python 3.13+ offers an experimental free-threaded (no-GIL) build; until it
> is mainstream, assume the GIL for CPU-bound decisions.

**`PY-CONC-02` (MUST NOT)** Don't run blocking calls or CPU-heavy work directly in
an `asyncio` event loop — it stalls all tasks. Offload via `run_in_executor`/
`asyncio.to_thread` (`GEN-CONC-14`). Don't mix blocking and async carelessly.

**`PY-CONC-03` (SHOULD)** Use `asyncio.TaskGroup` (3.11+) for structured
concurrency, propagate cancellation, and put timeouts on awaits (`asyncio.timeout`).

---

## 15a. Logging

See [general observability](../10-observability.md) for the cross-language model.
Python-specific:

**`PY-LOG-01` (SHOULD)** Use the `logging` module, not `print`, for diagnostic
output. `print` cannot be filtered by level, routed to handlers, or structured.

**`PY-LOG-02` (SHOULD)** Get a module-level logger with
`logger = logging.getLogger(__name__)` rather than logging on the root logger, so
output is namespaced and configurable per module.

**`PY-LOG-03` (SHOULD)** Use **lazy `%`-style** arguments, never f-strings, in log
calls so interpolation is deferred until (and unless) the record is emitted:

```python
# Bad: interpolates even when DEBUG is disabled; flagged by linters
logger.debug(f"loaded {len(rows)} rows for user {user_id}")

# Good: arguments interpolated lazily, only if the level is enabled
logger.debug("loaded %d rows for user %s", len(rows), user_id)
```

**`PY-LOG-04` (MUST)** Never log secrets, credentials, tokens, or full PII
(`GEN-SEC-39`). Configure logging once at the application entry point; libraries
should only obtain loggers, not configure handlers.

**`PY-LOG-05` (SHOULD)** Inside an `except` block, prefer `logger.exception(...)`
(or pass `exc_info=True`) so the traceback is captured with the message.

---

## 16. Packaging & environments

**`PY-PKG-01` (MUST)** Use an isolated virtual environment per project (`venv`,
uv, Poetry) — never install project deps into the system Python.

**`PY-PKG-02` (SHOULD)** Define project metadata and dependencies in
`pyproject.toml` (PEP 621); pin/lock for reproducibility (lockfile); separate
runtime vs dev dependencies.

**`PY-PKG-03` (SHOULD)** Structure importable code under a `src/` layout to avoid
accidentally importing the in-tree package instead of the installed one.

---

## 17. Security

See [security](../04-security.md). Python-specific:

**`PY-SEC-01` (MUST NOT)** Never `eval`/`exec`/`pickle.load` untrusted input; never
`yaml.load` untrusted YAML — use `yaml.safe_load` (`GEN-SEC-32`).
**`PY-SEC-02` (MUST)** Parameterize SQL (driver placeholders / ORM); never
f-string user input into queries (`GEN-SEC-03`).
**`PY-SEC-03` (MUST)** Avoid `subprocess(..., shell=True)` with untrusted input;
pass an args list with `shell=False` (`GEN-SEC-04`).
**`PY-SEC-04` (MUST)** Use the `secrets` module (not `random`) for tokens; use a
vetted KDF (argon2/bcrypt) for passwords (`GEN-SEC-10`, `GEN-SEC-22`).
**`PY-SEC-05` (SHOULD)** Run `pip-audit`/Safety and Bandit (SAST) in CI; pin/verify
dependencies to resist supply-chain attacks (`GEN-SEC-34`).

---

## 18. Testing

See [testing](../05-testing.md). Python-specific:

**`PY-TEST-01` (SHOULD)** Use **pytest**: plain `assert`, fixtures for setup,
`@pytest.mark.parametrize` for cases, `monkeypatch`/`unittest.mock` for doubles,
`tmp_path` for filesystem tests.
**`PY-TEST-02` (SHOULD)** Use **Hypothesis** for property-based testing of
invariant-rich logic (`GEN-TEST-17`); `coverage.py`/`pytest-cov` to find gaps (not
as a target — `GEN-TEST-19`).
**`PY-TEST-03` (SHOULD)** Keep tests deterministic: inject the clock/RNG, avoid
real network/time; use `freezegun`/fakes where needed.

---

## 19. Anti-patterns

- Mutable default arguments; late-binding closures in loops.
- Bare `except:`; over-broad `try`; swallowing exceptions with `pass`.
- `== None`, `== True`, `type(x) == T`, `if len(x) == 0`.
- Wildcard imports; imports not at top; unsorted import groups.
- `range(len(x))` indexing instead of `enumerate`/direct iteration.
- `%`/`.format` for new formatting instead of f-strings (outside logging).
- **f-strings (or eager `%`/`.format`) inside `logging` calls** — use lazy
  `%`-args.
- `print` for diagnostics instead of the `logging` module.
- Naming a `lambda` instead of using `def`.
- `str +=` in loops; floating-point `==`.
- Mutating a collection while iterating it.
- No type hints / pervasive `Any`.
- `eval`/`exec`/`pickle`/`yaml.load` on untrusted data; `shell=True`; `random` for
  secrets; f-string SQL.
- Installing into system Python; no lockfile.
- CPU-bound work on threads (GIL) or on the asyncio loop.

---

## Quick checklist

- [ ] Black/ruff-format + Ruff lint + mypy/pyright (strict) in pre-commit & CI.
- [ ] PEP 8 naming; no `l`/`O`/`I`; `Error`-suffixed exceptions; `__all__` for
      public API.
- [ ] 4-space indent; configured line length; PEP 8 spacing; wrap in parens, break
      before operators.
- [ ] Imports at top, grouped (stdlib/third-party/local), absolute, no wildcards.
- [ ] Pythonic idioms: `enumerate`/`zip`, comprehensions (readable), generators,
      f-strings, stdlib (`pathlib`, `collections`, `itertools`).
- [ ] EAFP with narrow `try`/specific excepts; `dict.get`/`getattr` where apt.
- [ ] Type hints on public APIs; modern syntax (`X | None`, built-in generics,
      `Protocol`); avoid `Any`.
- [ ] `@dataclass`(frozen/slots) / `enum` / Pydantic for data & validation.
- [ ] `is None`; truthiness for emptiness (with explicit `is not None` when needed);
      no `== True`; `isinstance`.
- [ ] Specific excepts, minimal `try`+`else`, exception chaining, no control flow in
      `finally`.
- [ ] `with`/context managers for resources; `encoding=` on `open`.
- [ ] PEP 257 docstrings (contract, not impl) on public APIs.
- [ ] No mutable default args; no late-binding closure bugs; `"".join` for strings;
      `math.isclose`/`Decimal`.
- [ ] Concurrency model matches workload (asyncio/threads/processes vs GIL); no
      blocking on the loop.
- [ ] `logging` (not `print`); module logger; lazy `%`-args (no f-strings in logs);
      no secrets logged.
- [ ] Options-heavy/boolean functions use keyword-only parameters.
- [ ] venv + `pyproject.toml` + lockfile + `src/` layout.
- [ ] No unsafe eval/pickle/yaml/shell/random; parameterized SQL; `secrets`;
      pip-audit/Bandit in CI.
- [ ] pytest + fixtures + parametrize + Hypothesis; deterministic (injected
      clock/RNG); coverage to find gaps.

---

## References

- PEP 8 — Style Guide for Python Code — https://peps.python.org/pep-0008/
- PEP 20 — The Zen of Python — https://peps.python.org/pep-0020/
- PEP 257 — Docstring Conventions — https://peps.python.org/pep-0257/
- PEP 484 / 526 / 604 / 695 — typing — https://peps.python.org/pep-0484/
- PEP 621 — project metadata in pyproject.toml — https://peps.python.org/pep-0621/
- Python docs, "Common Gotchas" (mutable defaults, late binding).
- Ruff — https://docs.astral.sh/ruff/ · Black — https://black.readthedocs.io/ ·
  mypy — https://mypy.readthedocs.io/
- Hypothesis — https://hypothesis.readthedocs.io/ · pytest — https://docs.pytest.org/
