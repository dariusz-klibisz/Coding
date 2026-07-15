# Terraform Review Checklist

> Use with [`../languages/terraform.md`](../languages/terraform.md) and
> [`general.md`](general.md).

- [ ] `terraform fmt`/`validate` clean; tflint + tfsec/Checkov pass; versions
      pinned (`TF-TOOL-*`).
- [ ] Module sources pinned to an exact version, not a mutable ref
      (`TF-MOD-03`).
- [ ] `for_each` used instead of `count` for addable/removable resource sets
      (`TF-MOD-04`).
- [ ] Every variable typed, described, validated where constrained; secrets
      marked `sensitive` (`TF-VAR-*`).
- [ ] Remote, locking state backend for shared state; `.tfstate` never
      committed; encrypted at rest (`TF-STATE-*`).
- [ ] `plan` reviewed before `apply`; destroy/recreate on stateful resources
      flagged for explicit sign-off (`TF-CHG-01/02`).
- [ ] No hardcoded secrets in `.tf`/`.tfvars`; least-privilege IAM
      (`TF-SEC-01/02`).
- [ ] Encryption-at-rest/no-public-exposure enforced by automated scan, not
      manual review alone (`TF-SEC-03`).
- [ ] Module tests present (`terraform test`/Terratest); security-policy
      scan gates CI (`TF-TEST-*`).
