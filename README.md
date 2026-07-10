# Coding Standards & State-of-the-Art Practices Reference

A curated, **citation-backed** reference library of software-engineering best
practices, coding guidelines, and state-of-the-art techniques. It covers
language-agnostic fundamentals — design principles, defensive programming, error
handling, security, testing, performance, concurrency, version control & CI,
documentation, observability, and tooling — and extends them with language-specific
guides and compact review checklists.

This repository is **documentation, not code.** There is no build or test suite —
only Markdown. It is designed to be **referenced from other repositories** while you
code, and to be equally useful to human engineers and AI coding agents. Every rule
states **what** to do and **why**, with **Pros/Cons** where a practice has genuine
trade-offs.

> **Deep entry point:** [`00-index.md`](00-index.md) contains the full navigation —
> the document maps, file-extension routing, the stable rule-ID scheme, the severity
> vocabulary, and the conflict-resolution priority order. Start there for anything
> beyond a quick orientation.

---

## File Map

### Root — general, language-agnostic (`01`–`12`)

| File | Rule IDs | Scope |
|---|---|---|
| [`00-index.md`](00-index.md) | — | Navigation, document maps, extension routing, rule-ID scheme, severity, priority order |
| [`01-principles.md`](01-principles.md) | `GEN-PRIN` | SOLID, DRY/KISS/YAGNI, coupling/cohesion, naming, immutability, SemVer, compatibility, tech debt |
| [`02-defensive-programming.md`](02-defensive-programming.md) | `GEN-DEF` | Fail-fast, assertions vs exceptions, design-by-contract, input validation, barricade pattern |
| [`03-error-handling.md`](03-error-handling.md) | `GEN-ERR` | Exceptions vs codes vs Result, retries, idempotency, resilience, error logging |
| [`04-security.md`](04-security.md) | `GEN-SEC` | OWASP Top 10, injection, authn/authz, crypto, secrets, deserialization, supply chain, DoS limits |
| [`05-testing.md`](05-testing.md) | `GEN-TEST` | Test pyramid/trophy, FIRST, AAA, doubles, property/fuzz, coverage, testability |
| [`06-performance.md`](06-performance.md) | `GEN-PERF` | Measure-first, complexity, profiling, allocation, caching, locality |
| [`07-concurrency.md`](07-concurrency.md) | `GEN-CONC` | Races/deadlocks, shared state, sync primitives, memory models, async/await |
| [`08-version-control-and-ci.md`](08-version-control-and-ci.md) | `GEN-VCS` | Commit hygiene, branching, PRs, review, CI/CD, quality gates |
| [`09-documentation-and-comments.md`](09-documentation-and-comments.md) | `GEN-DOC` | Self-documenting code, comment discipline, API docs, ADRs, docs for AI |
| [`10-observability.md`](10-observability.md) | `GEN-OBS` | Logging, structured logs, levels, metrics, tracing, health checks, alerting |
| [`11-tooling-and-automation.md`](11-tooling-and-automation.md) | `GEN-TOOL` | Formatters, linters, static analysis, pre-commit, CI design, dependency/secret automation |
| [`12-ai-agent-usage.md`](12-ai-agent-usage.md) | `GEN-AI` | Agent workflow, guardrails, priority order, when to ask |
| [`13-game-runtime-and-determinism.md`](13-game-runtime-and-determinism.md) | `GEN-GAME` | Frame budget, fixed timestep, object pooling, mobile constraints, seeded-RNG discipline, deterministic ordering, sim/presentation separation |

### Languages — extend the general docs

| File | Stack | Rule IDs |
|---|---|---|
| [`languages/embedded-c.md`](languages/embedded-c.md) | Embedded C (C99/C11) | `EC` |
| [`languages/csharp.md`](languages/csharp.md) | C# / .NET | `CS` |
| [`languages/python.md`](languages/python.md) | Python 3 (3.10+) | `PY` |
| [`languages/typescript.md`](languages/typescript.md) | TypeScript 5.x | `TS` |
| [`languages/vue.md`](languages/vue.md) | Vue 3 SFC + TS | `VUE` |
| [`languages/sql.md`](languages/sql.md) | SQL / PostgreSQL & PostGIS | `SQL` |
| [`languages/docker.md`](languages/docker.md) | Dockerfile + Docker Compose | `DOCKER` |
| [`languages/yaml.md`](languages/yaml.md) | YAML | `YAML` |
| [`languages/gdscript.md`](languages/gdscript.md) | GDScript / Godot 4.2+ | `GD` |
| [`languages/_template.md`](languages/_template.md) | Skeleton for new languages | — |

### Checklists — compact pre-delivery review gates

[`checklists/general.md`](checklists/general.md),
[`checklists/embedded-c.md`](checklists/embedded-c.md),
[`checklists/csharp.md`](checklists/csharp.md),
[`checklists/python.md`](checklists/python.md),
[`checklists/typescript.md`](checklists/typescript.md),
[`checklists/vue.md`](checklists/vue.md),
[`checklists/sql.md`](checklists/sql.md),
[`checklists/docker.md`](checklists/docker.md),
[`checklists/yaml.md`](checklists/yaml.md),
[`checklists/gdscript.md`](checklists/gdscript.md).

[`references.md`](references.md) collects the authoritative sources behind the rules.

The `00`–`12` numbering is a stable ordering; treat the filenames as durable anchors
for cross-references. Language docs are **self-numbered** so new languages can be
added without renumbering the root.

---

## How to Navigate

Start from the **task in front of you**, then load only what you need.

- **Orienting / finding a section** → [`00-index.md`](00-index.md), especially its
  document maps and *file-extension routing* table.
- **Editing a specific language** → load `languages/<lang>.md` + the matching
  `checklists/<lang>.md` (route by file extension):

  | Editing… | Load |
  |---|---|
  | `.py` | [`languages/python.md`](languages/python.md) + [`checklists/python.md`](checklists/python.md) |
  | `.ts` / `.tsx` | [`languages/typescript.md`](languages/typescript.md) + [`checklists/typescript.md`](checklists/typescript.md) |
  | `.vue` | [`languages/vue.md`](languages/vue.md) (+ TypeScript) + [`checklists/vue.md`](checklists/vue.md) |
  | `.cs` | [`languages/csharp.md`](languages/csharp.md) + [`checklists/csharp.md`](checklists/csharp.md) |
  | `.c` / `.h` | [`languages/embedded-c.md`](languages/embedded-c.md) + [`checklists/embedded-c.md`](checklists/embedded-c.md) |
  | `.sql` | [`languages/sql.md`](languages/sql.md) + [`checklists/sql.md`](checklists/sql.md) |
  | `Dockerfile*` / `docker-compose*.yml` | [`languages/docker.md`](languages/docker.md) + [`checklists/docker.md`](checklists/docker.md) |
  | `.yml` / `.yaml` | [`languages/yaml.md`](languages/yaml.md) + [`checklists/yaml.md`](checklists/yaml.md) |
  | `.gd` | [`languages/gdscript.md`](languages/gdscript.md) + [`checklists/gdscript.md`](checklists/gdscript.md) |

- **A cross-cutting concern** (security, testing, errors, concurrency, …) → load the
  matching root doc (`01`–`11`).
- **A final pre-delivery pass** → the matching file in [`checklists/`](checklists).

Each file is self-contained: it restates its scope and ends with a machine-scannable
**Quick Checklist** — read that first for an actionable summary.

---

## Conventions

- **Severity:** **MUST/MUST NOT** (non-negotiable), **SHOULD/SHOULD NOT** (strong
  default), **MAY/CONSIDER** (judgement call).
- **Rule IDs** are stable identifiers of the form `<PREFIX>-<TOPIC>-NN` (e.g.
  `PY-NAME-01`, `GEN-SEC-03`). Cite them when justifying a change. When a
  language-specific rule and a general rule conflict, **the language rule wins** for
  that language.
- **Conflict resolution** follows the priority order documented in
  [`00-index.md`](00-index.md): user requirement → safety/security/correctness →
  existing conventions & compatibility → official guidance → this reference →
  preference.

---

## For AI Coding Agents

This repo is structured for retrieval:

1. **Begin at [`00-index.md`](00-index.md).** Its document maps and routing tables map
   a topic or file extension to the exact file and rule-ID prefix to load.
2. **Load only what you need** — the relevant root doc(s) and/or `languages/<lang>.md`,
   plus a checklist for the final pass. Don't load everything.
3. **Cite rule IDs** when explaining or justifying a change so the user can trace the
   rationale.
4. **Sources live in [`references.md`](references.md).** Cite from there; do not invent
   citations.

Read [`12-ai-agent-usage.md`](12-ai-agent-usage.md) for the full agent workflow, and
[`AGENTS.md`](AGENTS.md) for content conventions and maintenance rules before editing
this repository.
