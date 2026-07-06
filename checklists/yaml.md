# YAML Review Checklist

> Use with [`../languages/yaml.md`](../languages/yaml.md) (full rationale and
> rule IDs) and [`general.md`](general.md).

- [ ] yamllint in CI and passing (`key-duplicates`, `truthy`, `indentation`, …)
      (`YAML-TOOL-01`).
- [ ] 2-space indentation; no tabs anywhere (`YAML-SYN-01`).
- [ ] No duplicate keys (silently override) (`YAML-SYN-02`).
- [ ] Type-ambiguous strings quoted: `"NO"`, `"yes"`, `"on"`, `"1.0"`, `"0777"`,
      `"1:30"` — Norway problem / 1.1-schema parsers (`YAML-SYN-03`).
- [ ] Lowercase `true`/`false`/`null` only; never `yes`/`no`/`on`/`off`
      (`YAML-SYN-04`).
- [ ] Block scalars: `|` for scripts/SQL (newlines preserved), `>` for prose
      (folded); `|-`/`>-` when the trailing newline matters (`YAML-SYN-05/06`).
- [ ] Plain scalars by default; double quotes only for escape sequences; single
      quotes otherwise (`YAML-STY-01`).
- [ ] Anchors/aliases only for genuinely shared config; no `<<:` merge keys
      unless every consuming parser supports them (never standardized)
      (`YAML-STY-02/03`).
- [ ] Lines < ~120 chars; long lists/env blocks broken vertically
      (`YAML-STY-04`).
- [ ] Trailing newline at EOF (`YAML-STY-05`).
- [ ] Nesting ≤ ~5 levels; flow vs block style consistent (`YAML-STY-06/07`).
- [ ] No secret literals; env-var references — and the consuming tool verified
      to actually interpolate `${VAR}` (`YAML-SEC-01`).
