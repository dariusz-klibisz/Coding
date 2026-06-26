# Documentation & Comments

> When and how to document code, write comments, and capture decisions — so that
> knowledge survives beyond the author's memory.

**Rule ID prefix:** `GEN-DOC`

---

## Table of contents

1. [Self-documenting code first](#1-self-documenting-code-first)
2. [Good comments vs bad comments](#2-good-comments-vs-bad-comments)
3. [What to comment](#3-what-to-comment)
4. [What NOT to comment](#4-what-not-to-comment)
5. [API / docstring documentation](#5-api--docstring-documentation)
6. [Keeping docs in sync](#6-keeping-docs-in-sync)
7. [Project-level documentation](#7-project-level-documentation)
8. [Architecture Decision Records](#8-architecture-decision-records-adrs)
9. [Documentation for AI agents](#9-documentation-for-ai-agents)
10. [Anti-patterns](#10-anti-patterns)
11. [Quick checklist](#quick-checklist)
12. [References](#references)

---

## 1. Self-documenting code first

**`GEN-DOC-01` (SHOULD)** Make the code itself the primary documentation. Clear
names, small functions, good types, and obvious structure convey intent better
than comments — because they cannot drift out of sync with the code.

**Why:** A comment is a separate artifact that the compiler never checks. When the
code changes and the comment doesn't, the comment becomes a lie — and a misleading
comment is *worse than none* (PEP 8 states this explicitly). Code that expresses
intent through naming and structure stays correct by construction.

```python
# Comment compensating for an unclear name (bad)
d = 30  # elapsed time in days

# Self-documenting (good)
elapsed_days = 30
```

**`GEN-DOC-02` (SHOULD)** Before writing a comment to explain *what* code does,
ask whether you can refactor (rename, extract a well-named function) so the
comment is unnecessary.

---

## 2. Good comments vs bad comments

The distinction (from *Clean Code*): comments should explain **why**, not **what**.
The code already says *what*; the reader needs the *why* that isn't in the code.

```c
// Bad (what): restates the code
i++;  // increment i

// Good (why): explains a non-obvious reason
i++;  // skip the BOM byte at the start of the UTF-8 stream
```

**`GEN-DOC-03` (SHOULD)** Reserve comments for information that cannot be expressed
in code: rationale, trade-offs, references (spec/RFC/ticket/paper), warnings, and
non-obvious constraints.

---

## 3. What to comment

**`GEN-DOC-04` (SHOULD)** Comment these high-value cases:

- **Why, not what:** the reason for a non-obvious decision or approach.
- **Intent behind tricky code:** why an unusual algorithm/optimization was chosen
  (ideally with the benchmark, per `GEN-PERF-18`).
- **Warnings/consequences:** "Do not reorder — hardware requires this sequence,"
  "O(n²) but n is bounded < 10."
- **Workarounds:** link the upstream bug/issue being worked around, so it can be
  removed when fixed.
- **Domain/business rules** that aren't obvious from code (cite the regulation,
  contract, or spec).
- **TODO/FIXME** with context and ideally a tracking link — not anonymous,
  context-free markers.
- **Public API contracts** (preconditions, postconditions, units, ownership,
  thread-safety) — see §5.

---

## 4. What NOT to comment

**`GEN-DOC-05` (SHOULD NOT)**

- **Redundant comments** restating the code (`i++; // add one to i`).
- **Commented-out code** — delete it; version control remembers. Dead commented
  code rots and confuses.
- **Misleading or outdated comments** — fix or remove them on sight.
- **Journal/changelog comments** in source (`// 2019-03 modified by X`) — that's
  what VCS history is for.
- **Noise comments** (`// constructor`, `// end if`).
- **Comments compensating for bad names/structure** — fix the code instead.

---

## 5. API / docstring documentation

**`GEN-DOC-06` (SHOULD)** Document every public API element (module, class,
function, method) with its contract: purpose, parameters, return value, errors/
exceptions thrown, side effects, units, and thread-safety where relevant.

**Why:** The public API is what others depend on; its documented contract is a
promise bound by SemVer (`GEN-PRIN-30`). Good API docs let consumers use the code
without reading its implementation — the essence of encapsulation.

**`GEN-DOC-07` (SHOULD)** Use the language's standard doc format and a generator so
docs live next to code and publish automatically:

| Language | Format | Tool |
|----------|--------|------|
| Python | docstrings (PEP 257); Google/NumPy/reST style | Sphinx, mkdocstrings |
| C# | XML doc comments (`///`) | DocFX, Sandcastle |
| TypeScript/JS | TSDoc / JSDoc | TypeDoc |
| C | Doxygen comments | Doxygen |

```python
def transfer(account_from: Account, account_to: Account, amount: Money) -> None:
    """Move funds between two accounts atomically.

    Args:
        account_from: Source account; must have sufficient balance.
        account_to: Destination account.
        amount: Positive amount to transfer.

    Raises:
        InsufficientFundsError: If ``account_from`` lacks ``amount``.
        ValueError: If ``amount`` is not positive.

    Note:
        Thread-safe: acquires both account locks in account-id order.
    """
```

**`GEN-DOC-08` (SHOULD)** Document *intent and contract*, not the implementation —
so the doc stays valid across refactors. Don't restate the signature in prose.

---

## 6. Keeping docs in sync

**`GEN-DOC-09` (SHOULD)** Treat documentation as part of the change: update docs in
the same commit/PR as the code they describe. Out-of-date docs erode trust in all
docs.

**`GEN-DOC-10` (CONSIDER)** Make docs verifiable where possible: doctests
(executable examples), generated API references (fail the build on broken
references), and example code compiled/tested in CI. Verified docs can't silently
rot.

---

## 7. Project-level documentation

**`GEN-DOC-11` (SHOULD)** Every project has, at minimum:

- **README:** what it is, why it exists, how to install/build/run/test, and where
  to go next. The single most important doc — it's the entry point.
- **CONTRIBUTING:** how to set up, code standards, PR process.
- **CHANGELOG:** notable changes per version (Keep a Changelog format; can be
  generated from Conventional Commits).
- **Architecture overview:** a high-level map (diagram + prose) of the major
  components and how they interact — orients newcomers fast.

**`GEN-DOC-12` (CONSIDER)** Follow the **Diátaxis** framework — separate docs into
**tutorials** (learning), **how-to guides** (tasks), **reference** (information),
and **explanation** (understanding). Mixing these confuses readers; each serves a
different need.

---

## 8. Architecture Decision Records (ADRs)

**`GEN-DOC-13` (CONSIDER)** Record significant architectural/technical decisions as
short, immutable ADRs: **context** (forces at play), **decision** (what was
chosen), **consequences** (trade-offs accepted), and **status** (proposed/
accepted/superseded).

**Why:** The most expensive lost knowledge is *why* a decision was made. Six
months later, "why did we use X instead of Y?" is unanswerable without an ADR,
leading teams to either cargo-cult or wastefully re-litigate decisions. ADRs make
the reasoning durable and let future readers safely revisit choices when context
changes.

**Pros:** preserves rationale; speeds onboarding; prevents repeated debates;
makes trade-offs explicit.
**Cons:** small writing overhead; only worth it for *significant* decisions (not
routine ones).

---

## 9. Documentation for AI agents

**`GEN-DOC-14` (CONSIDER)** Structure documentation so automated/AI tools can
consume it reliably:

- **Predictable structure** — consistent headings, stable section order, and
  stable identifiers (like the rule IDs in this library) that can be referenced.
- **Machine-scannable summaries** — a checklist or TL;DR section per document.
- **Self-contained context** — state assumptions and definitions explicitly rather
  than relying on tribal knowledge an agent can't access.
- **Co-located, conventional metadata** — agent-instruction files (e.g.
  `AGENTS.md`, tool config), well-formed docstrings, and typed signatures give
  agents reliable, parseable context.
- **Examples paired with expected output** — concrete, verifiable examples
  disambiguate intent better than prose.

**Why:** AI coding agents reason from the text and structure available to them.
Predictable, explicit, self-contained docs reduce hallucination and let agents
locate and apply the right guidance — the organizing principle of this very
library.

---

## 10. Anti-patterns

- **Comments that restate code** (`// loop over users`).
- **Commented-out code** left in the repo.
- **Stale/misleading comments and docs** contradicting the code.
- **Journal/changelog comments** in source files.
- **Documenting implementation instead of contract**, so docs break on refactor.
- **No README / no build-and-run instructions.**
- **Lost decision rationale** (no ADRs) → repeated debates and cargo-culting.
- **Docs in a separate system that's never updated** with the code.
- **Over-documenting trivial code** while under-documenting public contracts and
  the tricky parts.

---

## Quick checklist

- [ ] Code is self-documenting first (names, small functions, types); comments are
      a last resort for *what*.
- [ ] Comments explain *why*/intent/trade-offs/warnings/references, not *what*.
- [ ] No redundant, commented-out, journal, or misleading comments.
- [ ] Every public API has a contract docstring (params, returns, errors, side
      effects, units, thread-safety) in the standard format/tool.
- [ ] Docs describe contract/intent, not implementation; updated in the same PR as
      code; verified in CI where possible.
- [ ] README + CONTRIBUTING + CHANGELOG + architecture overview exist.
- [ ] Significant decisions captured as ADRs (context/decision/consequences).
- [ ] Docs structured predictably and explicitly enough for AI agents to consume.

---

## References

- Robert C. Martin, *Clean Code*, ch. 4 "Comments."
- Steve McConnell, *Code Complete*, ch. 32 "Self-Documenting Code."
- PEP 257 — Docstring Conventions — https://peps.python.org/pep-0257/
- Diátaxis documentation framework — https://diataxis.fr/
- Michael Nygard, "Documenting Architecture Decisions" (ADRs) — https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions
- Keep a Changelog — https://keepachangelog.com/
- Write the Docs — https://www.writethedocs.org/
