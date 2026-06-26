# Tooling, Formatting & Automation

> Make mechanical correctness automatic. Automation removes style debate, catches
> common defects early, and makes safe code the path of least resistance. This doc
> is the cross-language tooling baseline; each `languages/` file names the concrete
> tools for that stack.

**Rule ID prefix:** `GEN-TOOL`

---

## Table of contents

1. [Principle: let machines enforce mechanics](#1-principle-let-machines-enforce-mechanics)
2. [Formatters](#2-formatters)
3. [Linters](#3-linters)
4. [Static analysis](#4-static-analysis)
5. [Compiler/type-checker warnings](#5-compilertype-checker-warnings)
6. [EditorConfig](#6-editorconfig)
7. [Pre-commit hooks](#7-pre-commit-hooks)
8. [CI pipeline design](#8-ci-pipeline-design)
9. [Dependency automation](#9-dependency-automation)
10. [Secret scanning](#10-secret-scanning)
11. [Generated code](#11-generated-code)
12. [Suppressing tool warnings](#12-suppressing-tool-warnings)
13. [Baseline strategy for legacy code](#13-baseline-strategy-for-legacy-code)
14. [Anti-patterns](#14-anti-patterns)
15. [Quick checklist](#quick-checklist)
16. [References](#references)

---

## 1. Principle: let machines enforce mechanics

**`GEN-TOOL-01` (SHOULD)** Automate every objective, mechanical rule (formatting,
import order, lint patterns, type checks, security scans) so humans spend review
attention on design and correctness — the things machines can't judge
(`GEN-VCS-11`).

**Why:** Machines enforce consistency tirelessly and objectively. Automating
mechanics removes style friction from reviews, eliminates whole bug classes
pre-merge, and makes the standard executable rather than aspirational.

---

## 2. Formatters

**`GEN-TOOL-02` (SHOULD)** Use an automatic code formatter with its config checked
into the repository, and run it as a CI check. Formatting is not a matter of taste
to debate per-PR; pick the tool's defaults and move on.

**`GEN-TOOL-03` (SHOULD)** **Format only touched files** in normal work; do a
repo-wide reformat as a single, isolated, clearly-labeled commit (never mixed with
logic changes) so diffs and `git blame` stay meaningful.

Typical formatters (see language docs): `clang-format` (C/C++), `dotnet format` +
`.editorconfig` (C#), Black or `ruff format` (Python), Prettier (TS/Vue).

---

## 3. Linters

**`GEN-TOOL-04` (SHOULD)** Run a linter for bug-prone patterns and project
conventions. Linters make standards executable and catch issues before review.

**`GEN-TOOL-05` (SHOULD)** Start from the tool's **recommended rule set**, then
tune. Avoid drowning the team in low-value rules (warning fatigue). Treat warnings
as errors only **after** the baseline is clean (§13). Disable a rule locally only
with a written reason (§12).

---

## 4. Static analysis

**`GEN-TOOL-06` (SHOULD)** Use static analyzers for correctness, security,
nullability, resource-lifetime, and unsafe-construct detection — they find defects
that are hard to cover with tests.

**Why:** Deeper analysis (data-flow, taint, nullability) catches classes of bugs
(null derefs, leaks, injection sinks, UB) that unit tests rarely exercise. Budget
for triaging false positives; deep analysis can be slow, so place it appropriately
in the pipeline (§8).

Examples (see language docs): compiler warnings + clang-tidy/cppcheck + MISRA/CERT
tools (C), Roslyn/.NET analyzers + nullable reference types (C#), mypy/pyright/Ruff
(Python), `tsc` + vue-tsc + typed `typescript-eslint` rules (TS/Vue).

---

## 5. Compiler/type-checker warnings

**`GEN-TOOL-07` (SHOULD)** Enable strict compiler/type-checker warnings and treat
them as **errors for new code**. Compiler diagnostics are very high signal — they
catch undefined behavior, unsafe conversions, dead code, and portability issues
early.

**`GEN-TOOL-08` (SHOULD)** For legacy code, establish a baseline and **fail CI on
new warnings** rather than requiring all historical warnings fixed up front (§13).
Don't suppress warnings globally without a documented reason.

---

## 6. EditorConfig

**`GEN-TOOL-09` (SHOULD)** Commit an `.editorconfig` so charset, end-of-line, final
newline, indentation style/size, and trailing-whitespace trimming are consistent
across every editor and contributor. For C#, `.editorconfig` also carries analyzer
severities.

---

## 7. Pre-commit hooks

**`GEN-TOOL-10` (SHOULD)** Use pre-commit hooks for **fast, deterministic** checks
on staged files only: format, lint, secret-scan, and quick unit tests for changed
packages. Local hooks give feedback seconds after a mistake.

**`GEN-TOOL-11` (SHOULD NOT)** Don't put slow or flaky work in hooks — long
integration suites, network-dependent checks, or auto-fixes that touch unrelated
files. Hooks that frustrate developers get bypassed (`--no-verify`). Keep the heavy
gates in CI.

---

## 8. CI pipeline design

**`GEN-TOOL-12` (SHOULD)** Layer the pipeline **fastest/highest-signal first**, so
cheap checks fail fast before expensive ones run. A typical order:

1. Dependency install / cache restore.
2. Format check.
3. Lint / static analysis / type check.
4. Unit tests.
5. Build / package.
6. Integration tests.
7. Security / dependency scans.
8. Deployment validation.

**`GEN-TOOL-13` (SHOULD)** Keep CI fast and trustworthy (`GEN-VCS-17`): parallelize,
cache, shard tests, and make failures actionable. Quarantine and fix flaky
gates — flakiness erodes trust and creates bypass pressure.

---

## 9. Dependency automation

**`GEN-TOOL-14` (SHOULD)** Automate dependency updates and vulnerability detection
(Dependabot/Renovate + `npm audit`/`pip-audit`/etc.), but **review dependency
changes like code** — they carry security and behavior risk.

**`GEN-TOOL-15` (SHOULD)** Use **lockfiles**, review changelogs for major bumps,
run the full test suite on update PRs, and remove unused dependencies to shrink the
attack surface (`GEN-SEC-34`–`36`).

---

## 10. Secret scanning

**`GEN-TOOL-16` (SHOULD)** Scan commits and CI artifacts for secrets, both as a
pre-commit hook and server-side (gitleaks, trufflehog, platform secret scanning).

**`GEN-TOOL-17` (MUST)** If a secret leaks, **rotate it** — do not merely delete it
from history; assume it is already harvested (`GEN-SEC-24`). Keep test credentials
obviously fake and invalid.

---

## 11. Generated code

**`GEN-TOOL-18` (SHOULD)** Clearly separate generated code from handwritten code and
mark generated files. Manual edits to generated code are lost on regeneration and
create review noise.

**`GEN-TOOL-19` (SHOULD)** Check generated code in only when the project requires
it; otherwise generate in CI and verify it's current. Don't reformat generated code
unless the generator owns its formatting.

---

## 12. Suppressing tool warnings

**`GEN-TOOL-20` (SHOULD)** Suppress warnings **narrowly and with a reason**: one
rule, one line/scope, with a comment tied to a verified false positive or
intentional, documented exception.

**`GEN-TOOL-21` (SHOULD NOT)** Don't disable an analyzer project-wide because it
currently reports many issues — that hides real defects. Use a baseline instead
(§13).

```text
Good:  disable one rule for one line, with a reason linking the verified exception.
Bad:   disable the analyzer for the whole project to silence a noisy backlog.
```

---

## 13. Baseline strategy for legacy code

**`GEN-TOOL-22` (SHOULD)** Adopt tools on legacy repos via a **baseline**, not a
big-bang cleanup (which blocks adoption indefinitely):

1. Run the tool and capture the current violations as a baseline.
2. Triage and fix the **high-risk** findings immediately.
3. Baseline/suppress the rest *temporarily*.
4. **Fail CI on new violations** (ratchet — no backsliding).
5. Burn down the baseline gradually.

**Why:** Requiring every historical issue fixed before enabling a tool means the
tool never gets enabled. A baseline lets you stop the bleeding now and improve over
time.

---

## 14. Anti-patterns

- Manual formatting / per-PR style debates instead of a committed formatter.
- Repo-wide reformat mixed into a logic change (destroys diffs/blame).
- Drowning in low-value lint rules (warning fatigue); or ignoring all warnings.
- Slow/flaky pre-commit hooks that get bypassed.
- CI ordered slow-first, so cheap failures take ages to surface.
- Auto-merging dependency updates without review or tests.
- Deleting a leaked secret from history instead of rotating it.
- Hand-editing generated code; reformatting generated output.
- Project-wide analyzer suppression to silence a backlog.
- Blocking tool adoption on a full historical cleanup instead of baselining.

---

## Quick checklist

- [ ] Formatter committed + CI-checked; touched-files-only except isolated reformat
      commits.
- [ ] Linter from recommended set, tuned; warnings-as-errors after baseline clean.
- [ ] Static analysis for correctness/security/nullability/lifetime in CI.
- [ ] Strict compiler/type-checker warnings; new warnings fail CI.
- [ ] `.editorconfig` committed (charset/EOL/indent/trim, analyzer severities).
- [ ] Pre-commit hooks fast & deterministic (format/lint/secret-scan/quick tests).
- [ ] CI layered fastest-first; fast, cached, sharded; flaky gates quarantined.
- [ ] Dependency updates automated, lockfiled, reviewed, tested; unused deps removed.
- [ ] Secret scanning pre-commit + server-side; leaked secrets rotated.
- [ ] Generated code marked/separated; not hand-edited; current in CI.
- [ ] Suppressions narrow + justified; no project-wide disables.
- [ ] Legacy adoption via baseline + ratchet (fail on new violations).

---

## References

- EditorConfig — https://editorconfig.org/
- pre-commit framework — https://pre-commit.com/
- Conventional Commits (commit-message automation) — https://www.conventionalcommits.org/
- OWASP Dependency-Check — https://owasp.org/www-project-dependency-check/
- gitleaks — https://github.com/gitleaks/gitleaks
- Google Engineering Practices, "Code Review" — https://google.github.io/eng-practices/review/
