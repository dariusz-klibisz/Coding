# YAML (`.yml` / `.yaml`) — Coding Standards & State-of-the-Art Practices

> **Scope:** YAML as a configuration language (CI pipelines, Docker Compose,
> Kubernetes manifests, application config). Covers syntax and implicit-typing
> pitfalls, style, secrets, and tooling. Docker-Compose-specific semantics live
> in [docker.md](docker.md); this file covers the language itself.
> **Primary sources:** YAML 1.2.2 specification (yaml.org), yamllint
> documentation.
> **Relationship to general docs:** extends [general docs](../00-index.md) —
> especially [security](../04-security.md) and
> [tooling & automation](../11-tooling-and-automation.md). Where a general rule
> and a YAML rule conflict, the YAML rule wins for YAML files.

**Rule ID prefix:** `YAML`

---

## Table of contents

1. [Tooling & enforcement](#1-tooling--enforcement)
2. [Indentation & core syntax](#2-indentation--core-syntax)
3. [Implicit typing pitfalls (the Norway problem)](#3-implicit-typing-pitfalls-the-norway-problem)
4. [Scalars & quoting style](#4-scalars--quoting-style)
5. [Block scalars](#5-block-scalars)
6. [Anchors, aliases & merge keys](#6-anchors-aliases--merge-keys)
7. [Document structure & layout](#7-document-structure--layout)
8. [Secrets](#8-secrets)
9. [Anti-patterns](#9-anti-patterns)
10. [Quick checklist](#quick-checklist)
11. [References](#references)

---

## 1. Tooling & enforcement

**`YAML-TOOL-01` (SHOULD)** Run **yamllint** in CI on all YAML in the repository
(infra, CI pipelines, app config). It catches the failure modes YAML makes
silent: duplicate keys, tab indentation, truthy-ambiguous values (`yes`/`NO`),
inconsistent indentation, trailing whitespace, and missing final newline. Most
rules in this file map to a yamllint rule (`key-duplicates`, `truthy`,
`indentation`, `line-length`, `new-line-at-end-of-file`).

---

## 2. Indentation & core syntax

**`YAML-SYN-01` (MUST)** Indent with **2 spaces** per level. The YAML spec
forbids tabs **in indentation** — a stray tab produces a cryptic parse error
pointing at the wrong line. (Tabs are technically legal *inside scalar content*,
but avoid them there too for consistency and diff hygiene.)

**`YAML-SYN-02` (MUST NOT)** Never rely on duplicate keys. YAML parsers commonly
accept a duplicated mapping key and **silently keep only one value** (typically
the last) — a copy-paste error becomes an invisible config override. yamllint's
`key-duplicates` rule catches this (`YAML-TOOL-01`).

```yaml
# Bad — the first port mapping silently disappears
services:
  api:
    ports: ["8080:8080"]
    ports: ["9090:9090"]   # duplicate key: only this one survives
```

---

## 3. Implicit typing pitfalls (the Norway problem)

YAML resolves unquoted scalars to types by pattern matching, and **which
patterns apply depends on the parser's schema**. YAML 1.1-schema parsers
(PyYAML, libyaml, go-yaml v2 — still ubiquitous) resolve far more forms than
the YAML 1.2 core schema does. You usually cannot control which parser reads
your file, so write defensively.

**`YAML-SYN-03` (MUST)** Quote any value that a parser could interpret as a
boolean, number, or null when you mean a **string**: `"yes"`, `"no"`, `"on"`,
`"off"`, `"true"`, `"false"`, `"null"`, `"1.0"`.

**Why — the Norway problem:** under a YAML 1.1 schema, `country: NO` parses as
`country: false` (`NO` is a 1.1 boolean synonym, like `yes`/`on`/`off`). A YAML
1.2 core-schema parser yields the string `"NO"` — but quoting defensively is the
only behavior that is correct under *both*. The same applies to version
strings: `version: 1.0` is the float `1.0` (and `1.10` becomes `1.1`), so quote
`"1.0"`.

```yaml
# Bad — meaning depends on the parser
country: NO          # false under YAML 1.1 parsers
version: 1.0         # float, not the string "1.0"

# Good — a string under every schema
country: "NO"
version: "1.0"
```

> **Note (other 1.1 quirks, briefly):** YAML 1.1 treats leading-zero integers as
> **octal** (`0777` → 511) where YAML 1.2 requires the explicit `0o777` form and
> reads `0777` as decimal — quote zero-padded identifiers (`"0123"`). YAML 1.1
> also parses colon-separated numbers as **sexagesimal** (`1:30` → 90) — quote
> time-like values (`"1:30"`).

**`YAML-SYN-04` (MUST)** Write booleans and null in lowercase JSON-compatible
form: `true`, `false`, `null`. `True`/`TRUE` and `Null`/`NULL`/`~` are valid in
the YAML 1.2 core schema but non-idiomatic; `yes`/`no`/`on`/`off` are **YAML
1.1-only** forms whose meaning depends on the parser — never use them.

---

## 4. Scalars & quoting style

**`YAML-STY-01` (SHOULD)** Prefer **plain scalars** (unquoted strings) when the
value contains no special characters and cannot be misinterpreted as another
type (`YAML-SYN-03`). When quoting is needed: use **double quotes** only when
you need escape sequences (`"line1\nline2"`, `"\t"`); use **single quotes**
otherwise (no escape processing; `''` is the only escape, for a literal quote).

```yaml
name: plain is fine          # plain scalar — readable, minimal noise
country: 'NO'                # quoted to force a string; no escapes needed
banner: "col1\tcol2\n"       # double quotes only because \t and \n are needed
```

---

## 5. Block scalars

**`YAML-SYN-05` (SHOULD)** Choose the block-scalar style by whether newlines are
data: use `|` (**literal**) for scripts, SQL, and anything line-oriented —
newlines are preserved exactly. Use `>` (**folded**) for long prose — single
newlines are folded into spaces (blank lines still produce a newline), so a
paragraph can be wrapped in the source without changing the value.

**`YAML-SYN-06` (SHOULD)** Be deliberate about **chomping**: a block scalar
keeps its final newline by default; append `-` (`|-`, `>-`) to strip it. A
trailing newline inside an env-var value or a key is a classic hard-to-see bug.

```yaml
script: |            # literal: each command on its own line, trailing \n kept
  set -e
  make build
description: >-      # folded: wrapped source, single-line value, no trailing \n
  A long sentence that is wrapped in the source
  but becomes one line when parsed.
```

---

## 6. Anchors, aliases & merge keys

**`YAML-STY-02` (CONSIDER)** Use anchors (`&name`) and aliases (`*name`) to
deduplicate **genuinely shared** configuration — one definition, referenced
where reused. Don't use them to build a templating layer: past a handful of
anchors, readers can no longer see what a node actually contains, and
`yq`/tool-based generation or the consumer's native mechanisms (e.g. Compose
`extends`, Helm) are clearer.

```yaml
x-logging: &default-logging
  driver: json-file
  options: { max-size: "10m" }

services:
  api:
    logging: *default-logging
  worker:
    logging: *default-logging
```

**`YAML-STY-03` (SHOULD NOT)** Don't rely on `<<:` **merge keys** in files that
multiple tools might read. Merge keys were a YAML 1.1-era working-draft
extension that was **never standardized** — YAML 1.2 did not adopt them — so
support is parser-dependent. Docker Compose supports them; verify any other
toolchain before relying on them, and never use them as a substitute for proper
templating.

---

## 7. Document structure & layout

**`YAML-STY-04` (SHOULD)** Keep lines under ~120 characters. Break long
sequences and environment blocks vertically — one item per line diffs and
reviews far better than a wrapped flow collection.

**`YAML-STY-05` (SHOULD)** End every file with a trailing newline — POSIX
convention; avoids `\ No newline at end of file` noise in diffs.

**`YAML-STY-06` (SHOULD NOT)** Avoid deep nesting (more than ~5 levels).
Deeply nested YAML is hard to read and indentation errors become invisible.
Flatten the structure or split the file.

**`YAML-STY-07` (SHOULD NOT)** Don't mix flow style (`{key: value}`,
`[a, b]`) and block style haphazardly in one file. Pick block style as the
default; reserve flow style for short leaf collections (e.g. a one-line list of
ports) and apply it consistently.

---

## 8. Secrets

Builds on [security](../04-security.md).

**`YAML-SEC-01` (MUST NOT)** Never hardcode secrets (passwords, tokens, API
keys) as literals in YAML files — they end up in version control, backups, and
logs. Reference environment variables and inject values at deploy/run time.

```yaml
# Bad — credential committed to the repo
database:
  password: hunter2

# Good — value injected at runtime by the consuming tool
database:
  password: ${DB_PASSWORD}
```

> **Note:** `${VAR}` interpolation is **not part of YAML** — it is a feature of
> the consuming tool (Docker Compose, some CI systems). To a plain YAML parser,
> `${DB_PASSWORD}` is just a string. Verify that the tool reading the file
> actually performs interpolation; otherwise use its native secret mechanism
> (Kubernetes `Secret` references, CI secret variables, a vault).

---

## 9. Anti-patterns

| Anti-pattern | Why harmful | Fix |
|---|---|---|
| Tabs for indentation | Spec forbids them in indentation; cryptic parse errors | 2 spaces always (`YAML-SYN-01`) |
| Unquoted `NO`/`yes`/`on`/`1.0` meant as strings | Parsed as boolean/float under YAML 1.1 schemas (Norway problem) | Quote the value (`YAML-SYN-03`) |
| `YES`/`NO`/`ON`/`OFF` as booleans | 1.1-only synonyms; parser-dependent meaning | Lowercase `true`/`false` (`YAML-SYN-04`) |
| Duplicate keys | Silently overrides values | yamllint `key-duplicates` (`YAML-SYN-02`) |
| Excessive nesting (>5 levels) | Hard to read and maintain | Flatten; split files (`YAML-STY-06`) |
| Secrets as literals | Leaks via version control | Env-var references / secret store (`YAML-SEC-01`) |
| Mixing flow and block styles | Visual inconsistency; harder review | Block style by default (`YAML-STY-07`) |
| `<<:` merge keys across toolchains | Never standardized; parser-dependent | Anchors/aliases or the tool's native mechanism (`YAML-STY-03`) |
| `>` folded block for scripts/SQL | Folding collapses newlines to spaces — breaks line-oriented content | `\|` literal block (`YAML-SYN-05`) |
| Unintended trailing newline in values | Invisible; corrupts env vars and keys | `\|-` / `>-` chomping (`YAML-SYN-06`) |

---

## Quick checklist

- [ ] yamllint in CI on all YAML files (`YAML-TOOL-01`).
- [ ] 2-space indentation; no tabs (`YAML-SYN-01`).
- [ ] No duplicate keys (`YAML-SYN-02`).
- [ ] Strings that look like booleans/numbers/null quoted: `"NO"`, `"yes"`,
      `"1.0"`, `"0777"`, `"1:30"` (`YAML-SYN-03`).
- [ ] Lowercase `true`/`false`/`null`; never `yes`/`no`/`on`/`off`
      (`YAML-SYN-04`).
- [ ] Block scalars: `|` for line-oriented content, `>` for prose; `|-`/`>-`
      when the trailing newline must not be part of the value
      (`YAML-SYN-05/06`).
- [ ] Plain scalars preferred; double quotes only for escape sequences, single
      quotes otherwise (`YAML-STY-01`).
- [ ] Anchors/aliases only for genuinely shared config; no `<<:` merge keys
      unless every consumer is verified to support them (`YAML-STY-02/03`).
- [ ] Lines < ~120 chars; trailing newline at EOF; nesting ≤ ~5 levels; flow vs
      block style consistent (`YAML-STY-04/05/06/07`).
- [ ] No secret literals; env-var references (interpolation is a tool feature —
      verify the consumer) (`YAML-SEC-01`).

---

## References

- YAML specification 1.2.2 — https://yaml.org/spec/1.2.2/
- YAML 1.1 type repository (bool `y`/`yes`/`on` forms) —
  https://yaml.org/type/bool.html
- YAML merge key working draft (1.1-era, never standardized) —
  https://yaml.org/type/merge.html
- yamllint documentation — https://yamllint.readthedocs.io/
