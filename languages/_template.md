# <LANGUAGE> — Coding Standards & State-of-the-Art Practices

> **HOW TO USE THIS TEMPLATE**
> 1. Copy this file to `languages/<lang>.md`.
> 2. Replace every `<PLACEHOLDER>` and fill in every section. Keep the section
>    **order and headings identical** to existing language files so AI agents and
>    humans can navigate predictably.
> 3. Choose a unique 2–3 letter **Rule ID prefix** and use it consistently
>    (`<PREFIX>-<TOPIC>-NN`).
> 4. For every rule: state the rule, give the **why**, and add **Pros/Cons** (and
>    "when to use/avoid") **only where there is a genuine trade-off** — not for
>    hard rules.
> 5. Pair **Bad/Good** code examples for non-obvious rules.
> 6. Cite the language's **authoritative** style guide(s)/standards in References.
> 7. Add a row to the Languages table in the top-level `README.md`.
> 8. Delete this instruction block when done.

> **Scope:** <language version(s), runtime(s), target use cases>.
> **Primary sources:** <official style guide(s) / standards>.
> **Relationship to general docs:** extends [general docs](../00-README.md). Where a
> general rule and a `<LANGUAGE>` rule conflict, the `<LANGUAGE>` rule wins for
> `<LANGUAGE>`.

**Rule ID prefix:** `<PREFIX>`

---

## Table of contents

1. [Tooling & enforcement](#1-tooling--enforcement)
2. [Naming conventions](#2-naming-conventions)
3. [Formatting & layout](#3-formatting--layout)
4. [Language idioms & state-of-the-art features](#4-language-idioms--state-of-the-art-features)
5. [Type safety / static analysis](#5-type-safety--static-analysis)
6. [Error handling](#6-error-handling)
7. [Memory / resource management](#7-memory--resource-management)
8. [Concurrency / async](#8-concurrency--async)
9. [Security](#9-security)
10. [Testing](#10-testing)
11. [Anti-patterns](#11-anti-patterns)
12. [Quick checklist](#quick-checklist)
13. [References](#references)

---

## 1. Tooling & enforcement

<Formatter, linter, type checker, analyzers, build flags. The configs that make
standards automatic and CI-enforceable. Cite the canonical tools. Builds on
[tooling & automation](../11-tooling-and-automation.md).>

- **`<PREFIX>-TOOL-01` (MUST/SHOULD)** ...

---

## 2. Naming conventions

<A table mapping each construct to its casing/convention, per the official guide.>

| Construct | Convention | Example |
|-----------|------------|---------|
| ... | ... | ... |

- **`<PREFIX>-NAME-01` (SHOULD)** ... — *why* ...

---

## 3. Formatting & layout

<Indentation, line length, braces, blank lines, wrapping. Defer to the formatter
where one exists.>

- **`<PREFIX>-FMT-01` (SHOULD)** ... — *why* ...

---

## 4. Language idioms & state-of-the-art features

<The idiomatic, modern way to write this language. Pair Bad/Good examples. Add
Pros/Cons where a choice has trade-offs (e.g. style A vs style B).>

- **`<PREFIX>-IDIOM-01` (SHOULD)** ... — *why* ...

```text
# Bad

# Good
```

---

## 5. Type safety / static analysis

<Type-system features, strictness settings, how to avoid escape hatches. (Omit or
adapt for dynamically/loosely typed languages.)>

- **`<PREFIX>-TYPE-01` (SHOULD)** ... — *why* ...

---

## 6. Error handling

<Language's error model (exceptions/codes/Result), conventions, and pitfalls.
Builds on [error handling](../03-error-handling.md).>

- **`<PREFIX>-ERR-01` (SHOULD)** ... — *why* ...

---

## 7. Memory / resource management

<Allocation model, ownership/GC/RAII, deterministic cleanup mechanism, leaks.
Builds on [defensive programming](../02-defensive-programming.md) §11.>

- **`<PREFIX>-RES-01` (MUST/SHOULD)** ... — *why* ...

---

## 8. Concurrency / async

<Threading/async model, synchronization, the language's specific hazards & tools.
Builds on [concurrency](../07-concurrency.md).>

- **`<PREFIX>-CONC-01` (SHOULD)** ... — *why* ...

---

## 9. Security

<Language-specific security pitfalls and safe APIs. Builds on
[security](../04-security.md).>

- **`<PREFIX>-SEC-01` (MUST)** ... — *why* ...

---

## 10. Testing

<Idiomatic test frameworks, mocking, property-based/fuzz tooling. Builds on
[testing](../05-testing.md).>

- **`<PREFIX>-TEST-01` (SHOULD)** ... — *why* ...

---

## 11. Anti-patterns

<A scannable bullet list of concrete things to avoid in this language, each
traceable to a rule above.>

- ...

---

## Quick checklist

<Machine-scannable, actionable bullets summarizing every section. This is the
section AI agents and reviewers read first.>

- [ ] ...

---

## References

<Authoritative primary sources only: official style guides, language specs,
standards bodies, canonical books. Include URLs.>

- <Official style guide> — <url>
- ...
