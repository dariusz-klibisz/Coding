# AGENTS.md — Guidance for AI Coding Agents

This file tells AI coding agents how to read, search, and edit this repository.
Humans are welcome to read it too. For orientation, start with
[`README.md`](README.md) and the deep index at [`00-index.md`](00-index.md). For the
full behavioral workflow when applying these standards to real code, read
[`12-ai-agent-usage.md`](12-ai-agent-usage.md).

## What this repository is

- A **coding-standards knowledge base** — a citation-backed reference library of
  best practices, guidelines, and state-of-the-art techniques. There is **no
  application code, build, or test suite** here — only Markdown.
- It is meant to be **referenced from other repositories** while coding, not run.
  The value is in the reasoning: every rule states **what** and **why**, with
  **Pros/Cons** where there is a genuine trade-off.

## How to use it (when working in another repo)

1. **Route via [`00-index.md`](00-index.md) first.** Its document maps and
   *file-extension routing* table map a topic or file extension to the exact file
   and rule-ID prefix. **Load only what you need**, not the whole library.
   - Cross-cutting work → the relevant root doc(s) (`01`–`12`).
   - Language work → `languages/<lang>.md` (route by extension: `.py`→python,
     `.ts`→typescript, `.vue`→vue, `.cs`→csharp, `.c`/`.h`→embedded-c).
   - Final pre-delivery pass → the matching `checklists/<topic>.md`.
2. **Cite rule IDs** (e.g. `PY-LOG-03`, `GEN-SEC-03`) when explaining or justifying a
   change. When a language-specific rule and a general rule conflict, **the language
   rule wins** for that language.
3. **Resolve conflicts** in the documented priority order (see
   [`00-index.md`](00-index.md)): explicit user requirement → safety/security/
   correctness/regulatory → existing conventions & compatibility → official
   language/framework guidance → this reference → preference.
4. **Apply judgement, not mechanically.** This is a strong default, not a mandate
   that overrides the user, the project, or correctness. See
   [`12-ai-agent-usage.md`](12-ai-agent-usage.md) for the full guardrails.

## Structure & conventions

- Root docs are numbered `00`–`12`; `languages/` and `checklists/` hold the
  extensions and review gates (see the File Map in [`README.md`](README.md)). The
  numbering is a stable ordering; **do not renumber or rename files** without updating
  every cross-reference.
- [`00-index.md`](00-index.md) is the canonical index: document maps, extension
  routing, the rule-ID scheme, the severity vocabulary, and the conflict-resolution
  order.
- **Rule IDs** are stable identifiers of the form `<PREFIX>-<TOPIC>-NN`. General docs
  use `GEN-<AREA>`; language docs use a short prefix (`EC`, `CS`, `PY`, `TS`, `VUE`,
  `GO`, `RS`, `SWIFT`, `KT`, `DART`, etc. — see `00-index.md` for the full list).
  Treat existing IDs as durable; do not renumber or reuse them.
- **Severity vocabulary:** MUST/MUST NOT, SHOULD/SHOULD NOT, MAY/CONSIDER — use these
  exact keywords.
- Each file restates its scope at the top, carries a **Table of contents**, and ends
  with a machine-scannable **Quick Checklist**. Language docs are **self-numbered** so
  new languages can be added without renumbering the root.
- Cross-links use relative paths, e.g. `[general docs](../00-index.md)` and
  `[`12-ai-agent-usage.md`](12-ai-agent-usage.md)`.

## Rules for editing / extending

When modifying content in this repo:

1. **Preserve structure and heading levels.** Keep the per-file section order, the
   Table of contents, and the closing Quick Checklist intact so docs chunk cleanly.
2. **Keep rule IDs stable.** Don't renumber, reuse, or repurpose an existing ID. New
   rules get the next free number in their `<PREFIX>-<TOPIC>` series.
3. **Keep cross-links valid.** Many files link back to `00-index.md` and cite rule IDs
   across files. If you rename a file or reword a heading, update every link and the
   index tables that target it.
4. **Update the index and README.** When adding, removing, or restructuring content,
   reflect it in [`00-index.md`](00-index.md)'s document maps / routing tables and the
   File Map in [`README.md`](README.md).
5. **Add a new language with the template.** Copy
   [`languages/_template.md`](languages/_template.md), keep its section order, choose a
   unique rule-ID prefix, add rows to the routing tables, and (optionally) add
   `checklists/<lang>.md`.
6. **Cite real sources.** Authoritative references belong in
   [`references.md`](references.md). Do not invent citations or attribute claims to
   sources that don't support them.
7. **State trade-offs.** Add **Pros/Cons** and **when to use / when to avoid** where a
   practice has a genuine trade-off; state hard rules plainly without artificial
   trade-off sections.
8. **Match the existing tone.** Neutral, concise, technical. No marketing language, no
   emojis, no first-person voice.

## What not to do

- Do not add code, tooling, dependencies, or a build/test system — this is a docs-only
  repository.
- Do not reproduce copyrighted standards (MISRA, CERT, PEP text, etc.) wholesale;
  summarize and link instead.
- Do not break the index or leave dangling links / anchors.
- Do not renumber files or rule IDs, or convert the rationale-driven prose into a
  context-free rulebook.

## Quick checklist before committing a docs change

- [ ] File structure, Table of contents, and Quick Checklist preserved.
- [ ] Rule IDs stable; new rules use the next free number in their series.
- [ ] All internal links (to `00-index.md`, rule IDs, files) still resolve.
- [ ] [`00-index.md`](00-index.md) and [`README.md`](README.md) updated if structure
      or content changed.
- [ ] New claims backed by a source in [`references.md`](references.md).
- [ ] Trade-offs stated; tone neutral, concise, and consistent with the rest.
