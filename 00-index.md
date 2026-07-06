# Coding Standards & State-of-the-Art Practices — Index & Routing

> The canonical **navigation index** for this reference library: document maps,
> file-extension routing, the rule-ID scheme, the severity vocabulary, and the
> conflict-resolution priority order. For a short orientation, start with
> [`README.md`](README.md); for the full agent workflow, see
> [`12-ai-agent-usage.md`](12-ai-agent-usage.md).
>
> A curated, citation-backed reference library of software-engineering best
> practices, coding guidelines, and state-of-the-art techniques. Written to be
> equally useful to human engineers and AI coding agents.
>
> This library unifies and reconciles two prior reference sets into one
> authoritative source: it keeps stable rule IDs and severity keywords as the
> backbone, folds in every unique topic from both sources, resolves conflicting
> advice to verified best practice (with explanatory notes), and corrects the
> factual discrepancies that were found during the merge.

## Purpose

This library answers two questions for any coding decision:

1. **What** is the recommended practice?
2. **Why** is it recommended (and what are the trade-offs)?

Every rule includes a rationale. Where a practice has genuine trade-offs, a
**Pros / Cons** block and **When to use / When to avoid** guidance is provided.
Hard rules (e.g. "compare to `None` with `is`", "never log secrets") are stated as
rules without artificial trade-off sections.

---

## How AI agents should use this library

Start here, then **load only what you need**. See
[`12-ai-agent-usage.md`](12-ai-agent-usage.md) for the full agent workflow.

1. Read this index and pick the file(s) matching the task (use the **"Load when"**
   column and the **file-extension routing** table below).
2. Each file is **self-contained**: it restates its scope and ends with a
   machine-scannable **Quick Checklist** — read the checklist first for an
   actionable list.
3. Rules carry **stable IDs** (e.g. `PY-NAME-01`, `GEN-DEF-03`). Reference these
   IDs when explaining or justifying a change.
4. Root files (`01`–`12`) are language-agnostic. `languages/` files *extend or
   specialize* them; **when both apply, the language-specific rule wins** for that
   language.
5. For a final pre-delivery pass, use the matching file in `checklists/`.

### File-extension routing (for language work)

| You are editing… | Load |
|------------------|------|
| `.py` | [`languages/python.md`](languages/python.md) + [`checklists/python.md`](checklists/python.md) |
| `.ts` / `.tsx` | [`languages/typescript.md`](languages/typescript.md) + [`checklists/typescript.md`](checklists/typescript.md) |
| `.vue` | [`languages/vue.md`](languages/vue.md) (+ TypeScript) + [`checklists/vue.md`](checklists/vue.md) |
| `.cs` | [`languages/csharp.md`](languages/csharp.md) + [`checklists/csharp.md`](checklists/csharp.md) |
| `.c` / `.h` (firmware) | [`languages/embedded-c.md`](languages/embedded-c.md) + [`checklists/embedded-c.md`](checklists/embedded-c.md) |
| `.sql` (schemas, queries, migrations) | [`languages/sql.md`](languages/sql.md) + [`checklists/sql.md`](checklists/sql.md) |
| `Dockerfile*` | [`languages/docker.md`](languages/docker.md) + [`checklists/docker.md`](checklists/docker.md) |
| `docker-compose*.yml` / `compose.yaml` | [`languages/docker.md`](languages/docker.md) (+ YAML) + [`checklists/docker.md`](checklists/docker.md) |
| `.yml` / `.yaml` (other) | [`languages/yaml.md`](languages/yaml.md) + [`checklists/yaml.md`](checklists/yaml.md) |

---

## Document map (root — general, language-agnostic)

| File | Rule IDs | Topic | Load when… |
|------|----------|-------|------------|
| [`01-principles.md`](01-principles.md) | `GEN-PRIN` | SOLID, DRY/KISS/YAGNI, coupling/cohesion, naming, immutability, SemVer, compatibility contracts, tech debt | designing or refactoring structure/APIs |
| [`02-defensive-programming.md`](02-defensive-programming.md) | `GEN-DEF` | Fail-fast, assertions vs exceptions, design-by-contract, input validation, barricade pattern | hardening inputs / robustness |
| [`03-error-handling.md`](03-error-handling.md) | `GEN-ERR` | Exceptions vs codes vs Result, retries, idempotency, resilience, error logging | signaling/propagating/recovering from errors |
| [`04-security.md`](04-security.md) | `GEN-SEC` | OWASP Top 10, injection, authn/authz, crypto, secrets, deserialization, supply chain, DoS limits | any security-relevant change |
| [`05-testing.md`](05-testing.md) | `GEN-TEST` | Test pyramid/trophy, FIRST, AAA, doubles, property/fuzz, coverage, testability | writing or reviewing tests |
| [`06-performance.md`](06-performance.md) | `GEN-PERF` | Measure-first, complexity, profiling, allocation, caching, locality | optimizing (after measuring) |
| [`07-concurrency.md`](07-concurrency.md) | `GEN-CONC` | Races/deadlocks, shared-state, sync primitives, memory models, async/await, models | multithreaded/async code |
| [`08-version-control-and-ci.md`](08-version-control-and-ci.md) | `GEN-VCS` | Commit hygiene, branching, PRs, review, CI/CD, quality gates | git/PR/CI/release work |
| [`09-documentation-and-comments.md`](09-documentation-and-comments.md) | `GEN-DOC` | Self-documenting code, comment discipline, API docs, ADRs, docs for AI | writing comments/docs/ADRs |
| [`10-observability.md`](10-observability.md) | `GEN-OBS` | Logging, structured logs, levels, metrics, tracing, health checks, alerting | making code diagnosable in prod |
| [`11-tooling-and-automation.md`](11-tooling-and-automation.md) | `GEN-TOOL` | Formatters, linters, static analysis, pre-commit, CI design, dependency/secret automation, baselines | setting up/enforcing tooling |
| [`12-ai-agent-usage.md`](12-ai-agent-usage.md) | `GEN-AI` | Agent workflow, guardrails, priority order, when to ask | acting as an AI coding agent |

## Document map (languages — extend the general docs)

Language files use **their own internal section numbering** so new languages can be
added to `languages/` without renumbering anything in the root.

| File | Stack | Rule IDs | Primary standards |
|------|-------|----------|-------------------|
| [`languages/embedded-c.md`](languages/embedded-c.md) | Embedded C (C99/C11) | `EC` | MISRA C, SEI CERT C, Barr Group |
| [`languages/csharp.md`](languages/csharp.md) | C# / .NET (current LTS+) | `CS` | MS .NET conventions, Framework Design Guidelines |
| [`languages/python.md`](languages/python.md) | Python 3 (3.10+) | `PY` | PEP 8 / 20 / 257 / 484 / 621 |
| [`languages/typescript.md`](languages/typescript.md) | TypeScript 5.x | `TS` | TS Handbook, `strict` mode, ESLint/Prettier |
| [`languages/vue.md`](languages/vue.md) | Vue 3 SFC + TS | `VUE` | Official Vue Style Guide (A–D), Composition API |
| [`languages/sql.md`](languages/sql.md) | SQL / PostgreSQL 13+ & PostGIS 3 | `SQL` | PostgreSQL docs, PostGIS docs, PG wiki "Don't Do This" |
| [`languages/docker.md`](languages/docker.md) | Dockerfile (BuildKit) + Docker Compose v2 | `DOCKER` | Docker "Building best practices", Compose Specification |
| [`languages/yaml.md`](languages/yaml.md) | YAML 1.2 (config files) | `YAML` | YAML 1.2.2 spec, yamllint |
| [`languages/_template.md`](languages/_template.md) | — | — | Skeleton for adding a new language |

## Checklists (compact pre-delivery review gates)

| File | Scope |
|------|-------|
| [`checklists/general.md`](checklists/general.md) | Cross-cutting review gate (any change) |
| [`checklists/embedded-c.md`](checklists/embedded-c.md) | Embedded C |
| [`checklists/csharp.md`](checklists/csharp.md) | C# / .NET |
| [`checklists/python.md`](checklists/python.md) | Python |
| [`checklists/typescript.md`](checklists/typescript.md) | TypeScript |
| [`checklists/vue.md`](checklists/vue.md) | Vue |
| [`checklists/sql.md`](checklists/sql.md) | SQL / PostgreSQL / PostGIS / migrations |
| [`checklists/docker.md`](checklists/docker.md) | Docker / Compose |
| [`checklists/yaml.md`](checklists/yaml.md) | YAML |

[`references.md`](references.md) collects authoritative sources and authority notes.

---

## Severity vocabulary

| Term | Meaning |
|------|---------|
| **MUST / MUST NOT** | Non-negotiable. Violations are defects. |
| **SHOULD / SHOULD NOT** | Strong default. Deviate only with a documented reason. |
| **MAY / CONSIDER** | Optional improvement or judgement call. |

## Rule ID scheme

Rule IDs are stable identifiers of the form `<PREFIX>-<TOPIC>-<NN>`:

- General docs use `GEN-<AREA>` prefixes (`GEN-PRIN`, `GEN-DEF`, `GEN-ERR`,
  `GEN-SEC`, `GEN-TEST`, `GEN-PERF`, `GEN-CONC`, `GEN-VCS`, `GEN-DOC`, `GEN-OBS`,
  `GEN-TOOL`, `GEN-AI`).
- Language docs use a short language prefix (`EC`, `CS`, `PY`, `TS`, `VUE`, `SQL`, `DOCKER`, `YAML`).

Language rules cross-reference the general rules they specialize (e.g.
`PY-LOG-03` ties to `GEN-OBS-04`). Cite these IDs when justifying changes.

## Conflict-resolution priority order

When guidance conflicts, resolve in this order:

1. Explicit user or product requirement.
2. Safety, security, correctness, and regulatory constraints.
3. Existing repository conventions and public-API / persisted-data compatibility.
4. Language/framework official guidance.
5. This reference.
6. Personal preference.

Consistency matters, but do not preserve a harmful pattern just because it already
exists. If changing it would be broad or risky, isolate the fix and document the
reason.

## Conventions used in this library

- **Bad / Good** code blocks demonstrate the contrast for non-obvious rules.
- **Pros / Cons** blocks appear where a practice has real trade-offs.
- **Notes** flag reconciled differences between the two source sets (e.g. Vue
  `ref` vs `reactive`, TS `enum` usage, Python f-strings in logging).
- Citations link to authoritative primary sources (PEP, MS Learn, Vue.js, OWASP,
  SEI CERT, MISRA overview) so claims can be verified.

## Extending this library

To add a new language:

1. Copy [`languages/_template.md`](languages/_template.md) to
   `languages/<lang>.md`.
2. Fill in every section; keep the section order identical for predictability.
3. Choose a unique rule-ID prefix and use it consistently.
4. Add a row to the **languages** table and the **file-extension routing** table
   above, and (optionally) a `checklists/<lang>.md`.
5. Cite the language's authoritative style guide(s) in `references.md`.

No root files need renumbering — language docs are self-numbered.

## Scope note

This reference is not a substitute for official standards, project-specific rules,
product security review, or legal/regulatory requirements. Treat it as a strong
engineering baseline and adapt it deliberately. Standards evolve; re-verify
citations against primary sources before treating any specific rule as current.

## Last updated

2026-06-26.
