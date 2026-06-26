# Version Control, Code Review & CI/CD

> Practices for collaborating on code: commit hygiene, branching, reviews, and
> automated pipelines.

**Rule ID prefix:** `GEN-VCS`

---

## Table of contents

1. [Commit hygiene](#1-commit-hygiene)
2. [Conventional Commits](#2-conventional-commits)
3. [Branching strategies](#3-branching-strategies)
4. [Pull/merge requests](#4-pullmerge-requests)
5. [Code review](#5-code-review)
6. [Repository hygiene](#6-repository-hygiene)
7. [Continuous Integration](#7-continuous-integration)
8. [Continuous Delivery/Deployment](#8-continuous-deliverydeployment)
9. [Automated quality gates](#9-automated-quality-gates)
10. [Anti-patterns](#10-anti-patterns)
11. [Quick checklist](#quick-checklist)
12. [References](#references)

---

## 1. Commit hygiene

**`GEN-VCS-01` (SHOULD)** Make each commit a single, coherent, logical change that
builds and passes tests on its own (atomic commits).

**Why:** Atomic commits make history reviewable, bisectable (`git bisect` to find
the commit that introduced a bug), and revertable. A commit mixing a refactor, a
bug fix, and a formatting sweep is impossible to review or revert cleanly.

**`GEN-VCS-02` (SHOULD)** Write good commit messages: a concise imperative subject
(≤ ~50 chars, "Fix race in cache eviction"), a blank line, then a body explaining
**why** (not just what — the diff shows what). Reference issues/tickets.

```
Fix race condition in session cache eviction

Two threads could evict the same entry, causing a double-free of the
underlying buffer. Guard eviction with the existing cache lock and make
the check-and-remove atomic.

Fixes #1423
```

**`GEN-VCS-03` (SHOULD NOT)** Don't commit commented-out code, debug prints,
secrets, generated artifacts, or unrelated changes. Don't commit broken code to
shared branches.

---

## 2. Conventional Commits

**`GEN-VCS-04` (CONSIDER)** Adopt Conventional Commits
(`type(scope): description`) — `feat`, `fix`, `docs`, `refactor`, `test`,
`chore`, `perf`, etc., with `!`/`BREAKING CHANGE:` for breaks.

**Why:** A machine-readable commit convention enables automated changelog
generation and automated SemVer bumping (`feat` → minor, `fix` → patch,
breaking → major), tying commits directly to [SemVer](01-principles.md#17-semantic-versioning--api-stability).

**Pros:** automated releases/changelogs; consistent, scannable history.
**Cons:** requires discipline/tooling; overkill for tiny solo projects.

---

## 3. Branching strategies

| Strategy | Shape | Best for | Trade-offs |
|----------|-------|----------|------------|
| **Trunk-based development** | Short-lived branches merged to `main` ≥ daily; feature flags hide unfinished work | Teams practicing CI/CD; high throughput | Requires strong tests, flags, discipline; minimal isolation |
| **GitHub Flow** | `main` + short feature branches + PR | Most teams, continuous deploy | Simple; relies on CI gating |
| **Git Flow** | `develop`, `release`, `hotfix`, `feature` branches | Versioned releases, scheduled releases | Heavyweight; long-lived branches → merge hell |

**`GEN-VCS-05` (SHOULD)** Prefer short-lived branches and frequent integration.

**Why:** Long-lived branches accumulate divergence, causing painful merges and
delaying feedback ("merge hell"/integration debt). Trunk-based development —
correlated by DORA research with elite delivery performance — minimizes this by
integrating to a shared trunk continuously, using **feature flags** to keep
incomplete features dormant in production.

**`GEN-VCS-06` (SHOULD)** Keep the main/trunk branch always releasable (green
build, deployable). Protect it (required reviews + passing CI).

---

## 4. Pull/merge requests

**`GEN-VCS-07` (SHOULD)** Keep PRs small and focused. Small PRs get faster, higher-
quality reviews; large PRs get rubber-stamped.

**Why:** Review effectiveness drops sharply with size — defect-finding falls off
beyond a few hundred lines of diff. A focused PR is easier to reason about, test,
and revert.

**`GEN-VCS-08` (SHOULD)** Give the PR a clear description: what changed, why, how
to test, and any risk/rollout notes. Link the issue. Include screenshots for UI.

**`GEN-VCS-09` (CONSIDER)** Choose a merge style deliberately: **squash** for a
clean one-commit-per-PR history; **rebase** for linear history; **merge commit**
to preserve branch context. Be consistent.

---

## 5. Code review

**`GEN-VCS-10` (SHOULD)** Require review on changes to shared branches. Reviews
catch defects, spread knowledge, and enforce standards — their value is as much
about shared understanding as bug-finding.

**What reviewers should check:**

- **Correctness:** does it do what it claims? Edge cases, error paths, concurrency.
- **Design:** does it fit the architecture? Is there a simpler approach?
- **Readability:** will someone understand this in a year?
- **Tests:** adequate, meaningful coverage of the change.
- **Security & performance:** obvious risks (injection, secrets, hot-path costs).
- **Not style nitpicks** — those should be automated (formatter/linter), not
  burned on human attention.

**`GEN-VCS-11` (SHOULD)** Automate the objective stuff (formatting, lint, types,
tests) so reviews focus on design and correctness, which humans are uniquely good
at. Review comments should be kind, specific, and about the code, not the author.

**`GEN-VCS-12` (CONSIDER)** Review promptly (same day). A stalled review blocks the
author and grows merge conflicts; fast review keeps flow and small batch sizes.

**Pros of code review:** fewer defects, shared ownership, mentoring, consistency.
**Cons:** latency if slow; can become bikeshedding/gatekeeping if culture is poor
— mitigate with automation and small PRs.

---

## 6. Repository hygiene

**`GEN-VCS-13` (MUST)** Use `.gitignore` to keep build artifacts, dependencies,
local config, and secrets out of the repo. Never commit secrets (`GEN-SEC-24`);
add secret-scanning pre-commit hooks.

**`GEN-VCS-14` (SHOULD)** Commit lockfiles for applications (reproducible builds).
Keep large binaries out of git history (use LFS or artifact storage).

**`GEN-VCS-15` (SHOULD)** Maintain a `README` (what/why/how to build & run),
`CONTRIBUTING` guidelines, and a `LICENSE`. Record significant decisions as ADRs
(see [08-documentation-and-comments.md](09-documentation-and-comments.md)).

---

## 7. Continuous Integration

**`GEN-VCS-16` (SHOULD)** Every push runs an automated pipeline that builds, lints,
type-checks, and tests the change. Merge only on green.

**Why:** CI catches integration problems within minutes of introduction, when
they're cheapest to fix, and keeps the trunk releasable. The longer code goes
un-integrated, the more expensive the eventual integration.

**`GEN-VCS-17` (SHOULD)** Keep CI **fast** (parallelize, cache dependencies, run
the quick checks first, shard tests). A slow pipeline (>~10 min) erodes the
feedback loop and tempts people to bypass it.

**`GEN-VCS-18` (MUST)** Keep the build green. A persistently broken main build
blocks everyone; treat a red trunk as a stop-the-line, highest-priority event.

---

## 8. Continuous Delivery/Deployment

- **Continuous Delivery:** every green build is *releasable*; release is a
  one-click/manual decision.
- **Continuous Deployment:** every green build is *automatically* released to
  production.

**`GEN-VCS-19` (SHOULD)** Automate deployment with the same artifact promoted
across environments (build once, deploy many) and reproducible Infrastructure-as-
Code. Manual, hand-tuned deploys are error-prone and unrepeatable.

**`GEN-VCS-20` (SHOULD)** Make deployments safe and reversible: progressive
rollout (canary/blue-green), health checks, automated rollback, and feature flags
to decouple deploy from release.

**`GEN-VCS-21` (SHOULD)** Ensure observability — logs, metrics, traces, alerts —
so you can detect and diagnose problems after deploy. Deploying without monitoring
is flying blind.

**Pros of CD:** small batches → lower risk per release, faster feedback, quick
recovery (the DORA "elite" pattern: deploy often, fail rarely, recover fast).
**Cons:** demands strong automated testing, monitoring, and rollback; cultural and
infrastructure investment up front.

---

## 9. Automated quality gates

**`GEN-VCS-22` (SHOULD)** Enforce in the pipeline (and ideally as pre-commit hooks
for fast local feedback):

- **Formatter** (deterministic, non-negotiable) — ends style debate.
- **Linter / static analysis** — catches bug patterns and smells.
- **Type checker** — where applicable.
- **Tests** with a sensible coverage floor on new code (`GEN-TEST-19`).
- **Security scans** — dependency CVEs, secret scanning, SAST.
- **Build** must succeed with warnings-as-errors where practical.

**Why automate gates:** machines enforce consistency tirelessly and objectively,
freeing humans for judgement-based review and removing style/lint friction from
PRs.

---

## 10. Anti-patterns

- **Giant, mixed-purpose commits/PRs** that can't be reviewed or reverted.
- **Long-lived feature branches** → merge hell and delayed feedback.
- **Committing to a red build / leaving the build broken.**
- **Committing secrets, artifacts, or generated files.**
- **Rubber-stamp reviews** (LGTM without reading) and **bikeshedding** style in
  reviews instead of automating it.
- **Slow CI** that people learn to skip or ignore.
- **Manual, snowflake deployments** that can't be reproduced or rolled back.
- **Deploying without monitoring or a rollback plan.**
- **Force-pushing shared branches**, rewriting others' history.

---

## Quick checklist

- [ ] Commits are atomic, build green, and explain *why*; good messages.
- [ ] (Optional) Conventional Commits drive changelog/version automation.
- [ ] Short-lived branches; frequent integration; trunk always releasable & protected.
- [ ] PRs small and focused with clear descriptions and linked issues.
- [ ] Reviews required, prompt, kind, focused on design/correctness; style automated.
- [ ] `.gitignore` excludes artifacts/secrets; lockfiles committed; secret scanning on.
- [ ] CI builds/lints/type-checks/tests every change; fast; merge only on green.
- [ ] Build-once artifact promoted via automated, reproducible, reversible deploys.
- [ ] Progressive rollout + health checks + rollback + feature flags.
- [ ] Observability (logs/metrics/traces/alerts) in place.
- [ ] Quality gates (format, lint, types, tests, security) enforced in pipeline.

---

## References

- DORA / *Accelerate* (Forsgren, Humble, Kim) — the four key metrics and elite
  delivery practices — https://dora.dev/
- Jez Humble & David Farley, *Continuous Delivery*.
- Conventional Commits — https://www.conventionalcommits.org/
- Trunk-Based Development — https://trunkbaseddevelopment.com/
- Google Engineering Practices — Code Review — https://google.github.io/eng-practices/review/
- Pro Git — https://git-scm.com/book/
