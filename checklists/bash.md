# Bash / Shell Review Checklist

> Use with [`../languages/bash.md`](../languages/bash.md) and
> [`general.md`](general.md).

- [ ] ShellCheck clean; shfmt applied (`SH-TOOL-*`).
- [ ] Shebang present; `set -euo pipefail` at the top (or a documented
      deviation) (`SH-SAFE-01`).
- [ ] All variable expansions quoted; no `ls`-output parsing; `--` before
      filename args (`SH-QUOTE-*`).
- [ ] `mktemp` for temp files/dirs, cleaned up via `trap ... EXIT`
      (`SH-SAFE-03`).
- [ ] Every command whose failure matters is checked; errors on `stderr`
      with non-zero exit (`SH-ERR-*`).
- [ ] Function-local variables declared `local` (`SH-FUNC-01`).
- [ ] No `eval`/`bash -c` on untrusted input; no hardcoded/logged secrets
      (`SH-SEC-*`).
- [ ] Bats/shunit2 tests for non-trivial functions (`SH-TEST-01`).
