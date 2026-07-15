# Bash / POSIX Shell — Coding Standards & State-of-the-Art Practices

> **Scope:** Bash 4+ scripts (CI steps, build/deploy tooling, dev-environment
> automation). Where portability to non-Bash POSIX `sh` is required, that
> constraint is called out explicitly per rule.
> **Primary sources:** Google Shell Style Guide, ShellCheck's rule
> documentation, the Bash Reference Manual.
> **Relationship to general docs:** extends [general docs](../00-index.md). Where a
> general rule and a shell-scripting rule conflict, the shell rule wins for
> `.sh`/`.bash` files.

**Rule ID prefix:** `SH`

---

## Table of contents

1. [Tooling & enforcement](#1-tooling--enforcement)
2. [Naming conventions](#2-naming-conventions)
3. [Formatting & layout](#3-formatting--layout)
4. [Script safety idioms](#4-script-safety-idioms)
5. [Quoting & word splitting](#5-quoting--word-splitting)
6. [Error handling](#6-error-handling)
7. [Functions & scope](#7-functions--scope)
8. [Security](#8-security)
9. [Testing](#9-testing)
10. [Anti-patterns](#10-anti-patterns)
11. [Quick checklist](#quick-checklist)
12. [References](#references)

---

## 1. Tooling & enforcement

**`SH-TOOL-01` (MUST)** Run **ShellCheck** on every script in CI and as a
pre-commit hook — it catches quoting bugs, word-splitting hazards, and
portability issues that are otherwise found only by a script failing at
runtime, often on data the author didn't test with (`GEN-TOOL-04`).

**`SH-TOOL-02` (SHOULD)** Format with **shfmt** (consistent indentation,
`case` alignment) so formatting isn't a per-PR debate (`GEN-TOOL-02`).

**`SH-TOOL-03` (SHOULD)** Reach for a "real" scripting language (Python,
Go, etc.) once a shell script exceeds roughly a couple hundred lines,
needs real data structures, or needs robust error handling beyond exit
codes — shell is the right tool for gluing commands together, not for
general-purpose programming (`GEN-PRIN-02` KISS: pick the tool that keeps
the *solution* simple, not just the initial script).

---

## 2. Naming conventions

| Construct | Convention | Example |
|-----------|------------|---------|
| Function | `lower_snake_case` | `deploy_service`, `check_prereqs` |
| Local variable | `lower_snake_case` | `retry_count`, `tmp_dir` |
| Environment/exported variable | `UPPER_SNAKE_CASE` | `DEPLOY_ENV`, `API_TOKEN` |
| Readonly constant | `UPPER_SNAKE_CASE` + `readonly` | `readonly MAX_RETRIES=3` |
| Script file | `kebab-case.sh` | `deploy-service.sh` |

**`SH-NAME-01` (SHOULD)** Reserve `UPPER_SNAKE_CASE` for environment/exported
variables and constants; use `lower_snake_case` for ordinary local/script
variables — this convention lets a reader distinguish "this crosses a
process boundary" from "this is local" at a glance (Google Shell Style
Guide).

---

## 3. Formatting & layout

**`SH-FMT-01` (MUST)** Start every script with a shebang (`#!/usr/bin/env
bash` for Bash-specific scripts; `#!/bin/sh` only if the script is verified
POSIX-portable) as line 1.

**`SH-FMT-02` (SHOULD)** 2-space indentation (Google Shell Style Guide);
`then`/`do` on the same line as `if`/`for`/`while` (`if cond; then`).

**`SH-FMT-03` (SHOULD)** Put a comment block at the top of every non-trivial
script stating its purpose, required environment variables, and usage —
shell scripts have no type signature to document intent, so this comment is
the closest equivalent to a function contract (`GEN-DOC` §on comments).

---

## 4. Script safety idioms

**`SH-SAFE-01` (MUST)** Start every script with strict-mode flags:

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

- `-e` — exit immediately on an unhandled command failure (don't silently
  continue after an error, mirrors `GEN-DEF` fail-fast).
- `-u` — treat unset variables as an error (catches typo'd variable names
  instead of silently expanding to empty string).
- `-o pipefail` — a pipeline's exit status is the last **non-zero** command,
  not just the final one — without it, `cmd_that_fails | grep x` reports
  success because `grep`'s exit code (not `cmd_that_fails`'s) is what `$?`
  sees.
- `IFS=$'\n\t'` — narrows word-splitting to newline/tab, avoiding the
  default (space/tab/newline) silently splitting filenames containing
  spaces.

**Why this is non-negotiable for CI/deploy scripts:** without `set -e`, a
failed step (a failed build, a failed `curl`) can be silently swallowed and
the script proceeds to deploy/publish anyway — the shell equivalent of the
swallowed-error anti-pattern (`GEN-ERR-11`), except the default behavior
*is* to swallow unless you opt out.

**`SH-SAFE-02` (SHOULD)** Use `[[ ... ]]` (Bash's extended test) over
`[ ... ]` (POSIX test) in Bash-specific scripts — `[[` avoids word-splitting
and pathname-expansion pitfalls that make `[` easy to misuse, and supports
`&&`/`||`/pattern matching directly.

**`SH-SAFE-03` (SHOULD)** Use `mktemp` for temporary files/directories
(never a hardcoded `/tmp/myfile` — predictable temp paths are both a race
condition and a symlink-attack vector, §8), and clean them up with a `trap`
so cleanup runs even if the script exits early.

```bash
tmp_dir=$(mktemp -d)
trap 'rm -rf "$tmp_dir"' EXIT   # released on every exit path, incl. errors
```

---

## 5. Quoting & word splitting

**`SH-QUOTE-01` (MUST)** Double-quote every variable expansion
(`"$var"`, `"${array[@]}"`) unless word-splitting is explicitly intended and
commented as such — an unquoted expansion is the single most common source
of shell bugs (filenames with spaces breaking a loop, empty variables
vanishing an argument instead of passing an empty string).

```bash
# WRONG — breaks on filenames with spaces; vanishes if $file is empty
for f in $(ls $dir); do rm $f; done

# RIGHT — quoted, and avoiding ls-parsing entirely
for f in "$dir"/*; do rm -- "$f"; done
```

**`SH-QUOTE-02` (SHOULD)** Never parse `ls` output — use globs (`for f in
"$dir"/*`) or `find ... -print0 | while IFS= read -r -d '' f` for filenames
that may contain spaces/newlines. `ls`'s output format is for humans, not
scripts (ShellCheck SC2012).

**`SH-QUOTE-03` (SHOULD)** Use `--` before a filename argument that may
begin with `-` (`rm -- "$file"`), so a filename like `-rf` is not
misinterpreted as an option.

---

## 6. Error handling

Builds on [error handling](../03-error-handling.md).

**`SH-ERR-01` (MUST)** Check the exit status of every command whose failure
matters, either via `set -e` (§4) or an explicit `if ! cmd; then ...; fi` —
don't rely on a command's output looking right; check `$?`/the `if`
condition.

**`SH-ERR-02` (SHOULD)** Provide a meaningful error message and a non-zero
`exit` code on failure, written to `stderr` (`echo "error: ..." >&2`), not
`stdout` — mixing diagnostics into `stdout` corrupts output meant to be
piped/parsed by the next command.

**`SH-ERR-03` (SHOULD)** Use a `trap ... ERR` handler for centralized error
reporting (log the failing line/command) in scripts where a single
top-level catch-all is more maintainable than scattering `if ! cmd` checks
everywhere (`GEN-ERR-12`).

---

## 7. Functions & scope

**`SH-FUNC-01` (SHOULD)** Declare function-local variables with `local`
(or `local -r` for read-only locals) — without it, every variable is global
and a function can silently clobber a caller's variable of the same name.

**`SH-FUNC-02` (SHOULD)** Return data from a function via `stdout` (captured
with command substitution `result=$(my_func)`) and status via the exit
code — don't try to "return" a value through the exit code (it's limited to
0–255 and semantically means success/failure, not a value, `GEN-PRIN-15`
least astonishment).

---

## 8. Security

Builds on [security](../04-security.md). Shell-specific:

**`SH-SEC-01` (MUST)** Never interpolate untrusted input directly into a
command string passed to `eval`/`bash -c`/a subshell — this is command
injection (`GEN-SEC-04`). If dynamic command construction is unavoidable,
build an array and execute it directly (`"${cmd_array[@]}"`), never a
string that gets re-parsed by the shell.

**`SH-SEC-02` (MUST)** Never hardcode secrets in a script; read them from
environment variables sourced from a secrets manager or CI secret store,
and never `echo`/log them (`GEN-SEC-24`/`27`).

**`SH-SEC-03` (SHOULD)** Avoid predictable temp file paths (§4's `mktemp`
rule) — a predictable path in a world-writable directory is a symlink-race
vulnerability an attacker can exploit to redirect a write.

**`SH-SEC-04` (SHOULD)** Quote command substitutions used in a conditional
or comparison so an attacker-influenced value can't be re-interpreted as a
shell operator (`[[ "$val" == "expected" ]]`, not an unquoted comparison).

---

## 9. Testing

Builds on [testing](../05-testing.md). Shell-specific:

**`SH-TEST-01` (SHOULD)** Use **Bats** (Bash Automated Testing System) or
**shunit2** for unit-testing shell functions, asserting on exit codes and
`stdout`/`stderr` — treat shell functions as testable units, not just
"run it and eyeball the output" (`GEN-TEST-01`).

**`SH-TEST-02` (SHOULD)** ShellCheck itself functions as a static-analysis
test gate (§1) — run it as part of the same CI stage as unit tests, not as
an optional/manual step.

---

## 10. Anti-patterns

- Missing `set -euo pipefail` at the top of a CI/deploy script.
- Unquoted variable expansions (`$var` instead of `"$var"`).
- Parsing `ls` output instead of globs/`find -print0`.
- Predictable/hardcoded temp file paths instead of `mktemp`.
- `eval`/`bash -c` with untrusted, unsanitized input.
- Secrets echoed to logs or hardcoded in the script.
- Functions mutating caller-scope variables because `local` was omitted.
- Trying to "return" a data value through a numeric exit code.
- A shell script that has grown into a de-facto application (deep control
  flow, real data structures) instead of being rewritten in a general-purpose
  language.

---

## Quick checklist

- [ ] ShellCheck clean in CI + pre-commit; shfmt applied.
- [ ] Shebang present; `set -euo pipefail` (or justified deviation) at the
      top of every script.
- [ ] All variable expansions quoted; no `ls`-output parsing; `--` before
      filename arguments.
- [ ] `mktemp` for temp files/dirs, cleaned up via `trap ... EXIT`.
- [ ] Every command whose failure matters is checked; errors go to `stderr`
      with a non-zero exit code.
- [ ] Function-local variables declared `local`.
- [ ] No `eval`/`bash -c` on untrusted input; no hardcoded/logged secrets.
- [ ] Bats/shunit2 tests for non-trivial functions.
- [ ] Script hasn't outgrown shell — genuinely complex logic moved to a
      general-purpose language.

---

## References

- Google Shell Style Guide — https://google.github.io/styleguide/shellguide.html
- ShellCheck wiki (rule-by-rule rationale) — https://www.shellcheck.net/wiki/
- GNU Bash Reference Manual — https://www.gnu.org/software/bash/manual/bash.html
- Bats-core — https://bats-core.readthedocs.io/
- shfmt — https://github.com/mvdan/sh
