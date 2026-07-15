# AI Coding Agent Usage Guide

> How an AI coding agent should use this library and behave when editing a real
> repository. The guidance elsewhere is *what* good code looks like; this file is
> *how to act* so an agent applies it safely without doing harm.

**Rule ID prefix:** `GEN-AI`

---

## Table of contents

1. [Primary rule](#1-primary-rule)
2. [Priority order](#2-priority-order)
3. [How to load this library](#3-how-to-load-this-library)
4. [Before editing](#4-before-editing)
5. [During editing](#5-during-editing)
6. [After editing](#6-after-editing)
7. [Handling conflicts with local style](#7-handling-conflicts-with-local-style)
8. [Defensive guardrails for agents](#8-defensive-guardrails-for-agents)
9. [Review checklist for AI-generated code](#9-review-checklist-for-ai-generated-code)
10. [When to ask a question](#10-when-to-ask-a-question)

---

## 1. Primary rule

**`GEN-AI-01` (MUST)** Optimize for the user's goal, repository correctness, and
maintainability. **Do not apply rules from this library mechanically** when doing
so would increase risk, break compatibility, or fight established local
conventions. This reference is a strong default, not a mandate that overrides
judgement, the user, or the project.

---

## 2. Priority order

**`GEN-AI-02` (MUST)** When guidance conflicts, resolve in this order (same order
the whole library uses):

1. Explicit user instruction.
2. Build, test, security, safety, and regulatory requirements.
3. Existing codebase architecture and conventions.
4. Public API and persisted-data compatibility (`GEN-PRIN-32`).
5. Language/framework official guidance.
6. This reference.
7. Personal or generic preference.

Consistency matters, but don't preserve a *harmful* pattern just because it exists.
If fixing it is broad or risky, isolate the fix and document the rest as follow-up.

---

## 3. How to load this library

**`GEN-AI-03` (SHOULD)** Route via [`00-index.md`](00-index.md) first: it has an
index table mapping topic → file → rule-ID prefix → "load when" trigger. Load only
the documents relevant to the current task instead of everything.

- For **cross-cutting** work, load the relevant root doc(s) (`01`–`12`).
- For **language-specific** work, load `languages/<lang>.md` (route by file
  extension: `.py` → python, `.ts` → typescript, `.vue` → vue, `.cs` → csharp,
  `.c`/`.h` → embedded-c).
- For a **final pre-delivery pass**, load the matching `checklists/<topic>.md`.

**`GEN-AI-04` (SHOULD)** Cite **rule IDs** (e.g. `PY-LOG-03`, `GEN-SEC-03`) when
explaining or justifying a change, so the user can trace the rationale. When a
language-specific rule and a general rule conflict, the language rule wins for that
language.

---

## 4. Before editing

**`GEN-AI-05` (SHOULD)** Establish ground truth before changing anything:

- Inspect the repository structure before assuming frameworks, package managers, or
  style.
- Search for existing implementations before introducing new patterns.
- Read neighboring code and tests to learn local conventions.
- Identify the project's formatters, linters, analyzers, and test commands.
- Prefer the **smallest correct change**; preserve unrelated user changes.

---

## 5. During editing

**`GEN-AI-06` (SHOULD)** Keep changes focused and safe:

- Stay on the requested task; follow local naming and layout conventions.
- Don't add abstractions unless they remove real duplication or encode a domain
  concept (`GEN-PRIN-03`).
- Prefer direct, readable control flow over clever one-liners.
- Validate untrusted input at boundaries (`GEN-DEF-02`); preserve error context;
  don't swallow exceptions (`GEN-ERR-11`).
- Don't add hidden global state; don't introduce secrets, credentials, or
  environment-specific paths.
- Add comments only for non-obvious decisions, invariants, or hazards
  (`GEN-DOC-03`).

---

## 6. After editing

**`GEN-AI-07` (SHOULD)** Verify and report:

- Run the **narrowest meaningful** verification first, then broaden if the change
  touches shared code or behavior.
- Inspect the generated diff before finalizing.
- State which tests/checks were run, and which **could not** be run.
- Document any residual risk when verification is incomplete.

---

## 7. Handling conflicts with local style

**`GEN-AI-08` (SHOULD)** When the repo's style conflicts with this reference:

- Follow the **repository** for localized edits; don't reformat unrelated code.
- If the local style is *unsafe*, fix only the unsafe portion needed for the task.
- If a broad migration would be needed, ask permission or record it as a follow-up
  rather than silently rewriting (`GEN-TOOL-22` baseline mindset).

---

## 8. Defensive guardrails for agents

**`GEN-AI-09` (MUST)** AI agents make predictable mistakes. Apply these guardrails:

- **Don't invent APIs** — verify names and signatures in the code or official docs.
- **Don't assume a nullable value is present** — check types and call paths.
- **Don't assume async work is awaited** — inspect promise/task lifetimes
  (`GEN-CONC-13`).
- **Don't hand-edit generated code** (`GEN-TOOL-18`).
- **Don't update test snapshots** unless behavior was intentionally changed.
- **Don't add compatibility shims** without a concrete compatibility need.
- **Don't convert a whole file/codebase to a preferred paradigm** inside a targeted
  fix.

---

## 9. Review checklist for AI-generated code

- [ ] Does the change directly satisfy the user request?
- [ ] Is it minimal and localized?
- [ ] Does it follow repository conventions?
- [ ] Are inputs validated at trust boundaries?
- [ ] Are errors handled at the correct level, with context preserved?
- [ ] Is async/concurrent behavior safe (awaited, cancellable, bounded)?
- [ ] Is resource lifetime explicit (disposed/closed/released)?
- [ ] Are tests added/updated where behavior changed?
- [ ] Were formatting/linting/type checks respected?
- [ ] Were unrelated files left untouched?
- [ ] No secrets, hardcoded paths, or hidden global state introduced?

---

## 10. When to ask a question

**`GEN-AI-10` (SHOULD)** Ask **one concise question** when:

- The requested behavior is ambiguous and reasonable implementations differ in
  user-visible outcome.
- A change would break a public API, persisted data, or an existing workflow
  (`GEN-PRIN-32`).
- Required credentials, environments, or generated assets are unavailable.
- Unexpected concurrent user changes directly conflict with the task.

**`GEN-AI-11` (SHOULD NOT)** Don't ask permission for routine implementation steps
that the user's request clearly implies. Bias toward doing the obvious work and
reporting it, not toward asking.

---

## References

- This library's [`00-index.md`](00-index.md) — routing, rule-ID scheme, and
  conflict-resolution order.
- [`09-documentation-and-comments.md`](09-documentation-and-comments.md) §9 —
  writing docs that AI agents can consume reliably.
