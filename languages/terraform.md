# Terraform / HCL — Coding Standards & State-of-the-Art Practices

> **Scope:** Terraform (HCL2) configuration for infrastructure-as-code:
> resources, modules, providers, and state management. Covers the coding-
> level discipline of writing and organizing `.tf` files; cloud-architecture
> decisions (which resources to provision, multi-region strategy) are outside
> this library's scope.
> **Primary sources:** HashiCorp Terraform documentation, the Terraform
> Style Guide, `tflint`/`checkov`/`tfsec` rule documentation.
> **Relationship to general docs:** extends [general docs](../00-index.md),
> especially [security](../04-security.md) and
> [version control & CI](../08-version-control-and-ci.md). Where a general
> rule and a Terraform rule conflict, the Terraform rule wins for `.tf` files.

**Rule ID prefix:** `TF`

---

## Table of contents

1. [Tooling & enforcement](#1-tooling--enforcement)
2. [Naming conventions](#2-naming-conventions)
3. [Formatting & layout](#3-formatting--layout)
4. [Module design](#4-module-design)
5. [Variables & outputs](#5-variables--outputs)
6. [State management](#6-state-management)
7. [Change safety](#7-change-safety)
8. [Security](#8-security)
9. [Testing](#9-testing)
10. [Anti-patterns](#10-anti-patterns)
11. [Quick checklist](#quick-checklist)
12. [References](#references)

---

## 1. Tooling & enforcement

**`TF-TOOL-01` (MUST)** Format every file with `terraform fmt` (canonical,
non-configurable) and validate with `terraform validate` in CI
(`GEN-TOOL-02`).

**`TF-TOOL-02` (SHOULD)** Lint with **tflint** (provider-aware correctness
rules, e.g. deprecated arguments, invalid instance types) and scan with a
security-focused static analyzer (**tfsec** or **Checkov**) for
misconfigurations (public S3 buckets, unencrypted volumes, overly broad
IAM) before `apply` (`GEN-SEC-43`).

**`TF-TOOL-03` (MUST)** Pin the Terraform CLI version (`required_version`)
and every provider version (`required_providers` with a version constraint)
so `plan`/`apply` is reproducible across machines and time (`GEN-VCS-14`).

**`TF-TOOL-04` (SHOULD)** Run `terraform plan` in CI on every pull request
and require a human review of the plan output before `apply` — the plan is
the diff of a change that can delete production infrastructure, and
deserves the same review rigor as a code diff (`GEN-VCS-10`).

---

## 2. Naming conventions

| Construct | Convention | Example |
|-----------|------------|---------|
| Resource/data block name | `snake_case`, descriptive, no repeated type | `resource "aws_s3_bucket" "logs"` (not `"aws_s3_bucket_logs"`) |
| Variable / output | `snake_case` | `instance_count`, `vpc_id` |
| Module directory | `kebab-case` | `modules/network-vpc` |
| Local value | `snake_case` | `common_tags` |

**`TF-NAME-01` (SHOULD)** Don't repeat the resource type in its local name
(`aws_instance.web`, not `aws_instance.web_instance`) — the type is already
part of the fully-qualified reference (`GO-NAME-02`'s stutter rule, applied
to HCL).

**`TF-NAME-02` (SHOULD)** Name the "primary" or only resource of a kind in a
module/file `this` or `main` only when genuinely singular and unambiguous;
otherwise use a descriptive name — a repo-wide sea of resources all named
`this` makes cross-references (`aws_instance.this.id`) unreadable at scale.

---

## 3. Formatting & layout

**`TF-FMT-01` (MUST)** 2-space indentation, aligned `=` signs within a
block — this is exactly what `terraform fmt` produces; never hand-format
against it.

**`TF-FMT-02` (SHOULD)** Organize a module's files by convention:
`main.tf` (resources), `variables.tf` (inputs), `outputs.tf` (outputs),
`versions.tf` (provider/version constraints) — predictable file names let a
reader navigate any module the same way.

**`TF-FMT-03` (SHOULD)** Order block arguments consistently: `count`/
`for_each` (meta-arguments) first, then required arguments, then optional
arguments, then nested blocks, then lifecycle/`depends_on` — matches
`terraform fmt`'s canonical vertical alignment and the style the docs
themselves use.

---

## 4. Module design

**`TF-MOD-01` (SHOULD)** Extract a **module** once the same resource group
is instantiated more than once (Rule of Three, `GEN-PRIN-04`) — not before,
to avoid a premature abstraction over infrastructure shapes that later turn
out to differ per environment (`GEN-PRIN-03` YAGNI).

**`TF-MOD-02` (SHOULD)** Give every module a narrow, single responsibility
(a VPC module provisions networking, not networking + compute + DNS) —
mirrors Separation of Concerns (`GEN-PRIN-05`); a module that does
everything can't be reused or tested independently.

**`TF-MOD-03` (SHOULD)** Pin module sources to an exact version (a Git tag,
a registry version constraint) — never track a mutable ref (`main`,
`HEAD`) for a module dependency, or a producer's fix/break silently changes
every consumer's next `plan` (`GEN-PRIN-32` compatibility contracts).

```hcl
# Bad — floats with upstream changes, breaking reproducibility
module "vpc" {
  source = "github.com/example/tf-vpc"
}

# Good — pinned to an exact, reviewed version
module "vpc" {
  source  = "github.com/example/tf-vpc"
  version = "3.4.0"
}
```

**`TF-MOD-04` (SHOULD)** Use `for_each` over `count` for a set of similar
resources whose members can be added/removed individually — `count`-indexed
resources shift index on removal from the middle of the list, causing
Terraform to plan destroy/recreate for unrelated resources that merely
shifted position.

---

## 5. Variables & outputs

**`TF-VAR-01` (MUST)** Declare a `type` and `description` for every input
variable — an untyped variable accepts silently-wrong values, and an
undocumented one forces the next engineer to read the resource body to
learn what it does (`GEN-DOC` self-documenting code, applied to infra).

**`TF-VAR-02` (SHOULD)** Add `validation` blocks for variables with a
constrained valid range/format (an environment name, a CIDR block) so a
bad value fails at `plan` time with a clear message, not mid-`apply` against
a cloud API (`GEN-DEF-02` validate at the boundary).

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment name."
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod."
  }
}
```

**`TF-VAR-03` (SHOULD)** Mark variables carrying secrets `sensitive = true`
so Terraform redacts them from `plan`/`apply` console output and state
diffs shown in CI logs (`GEN-SEC-27`) — this reduces exposure but does not
replace proper secrets management (§8).

**`TF-VAR-04` (SHOULD)** Expose only the outputs consumers actually need
(encapsulation, `GEN-PRIN-14`) — don't output a module's entire internal
resource object by default; that couples every consumer to the module's
internals and blocks safe internal refactors.

---

## 6. State management

**`TF-STATE-01` (MUST)** Use a remote, locking backend (S3+DynamoDB, Terraform
Cloud, GCS with locking, Azure Storage) for any state shared by more than
one person — local state files invite concurrent-apply corruption and have
no history/locking (mirrors `GEN-CONC-05`'s check-then-act hazard, applied
to infrastructure changes).

**`TF-STATE-02` (MUST NOT)** Never commit a `.tfstate` file to version
control — it routinely contains secrets in plaintext (database passwords,
private keys read back from providers) even for resources that don't
appear to store secrets (`GEN-SEC-24`).

**`TF-STATE-03` (SHOULD)** Split state by blast radius (a state file per
environment, and per major system boundary) rather than one monolithic
state for an entire organization — a single giant state makes every
`apply` slower, raises lock contention, and means a single corrupted state
file can block all infrastructure changes everywhere.

**`TF-STATE-04` (SHOULD)** Enable state file encryption at rest (backend-
native encryption, e.g. S3 SSE) and restrict state-backend access to the
principals that actually run Terraform (`GEN-SEC` §13 secure defaults).

---

## 7. Change safety

**`TF-CHG-01` (MUST)** Never run `apply` without first reviewing the
corresponding `plan` output — an unreviewed `apply` can silently destroy
and recreate a resource (e.g. after a change to an argument that forces
replacement) with no warning beyond the plan diff itself.

**`TF-CHG-02` (SHOULD)** Treat any planned `-/+` (destroy and recreate) on a
stateful resource (a database, a persistent volume) as requiring explicit
sign-off, not an automatic-merge CI gate — data loss from an automated
destroy/recreate is generally unrecoverable (`GEN-VCS-20` progressive/
reversible deploys, applied to infrastructure).

**`TF-CHG-03` (SHOULD)** Use `terraform state mv`/`import`/`removed` blocks
(not manual state file editing) when refactoring resource addresses or
adopting existing infrastructure — hand-editing `.tfstate` JSON is
error-prone and unsupported.

**`TF-CHG-04` (CONSIDER)** Use policy-as-code (Sentinel, OPA/Conftest) to
enforce organizational guardrails (allowed instance types, mandatory tags,
region restrictions) as an automated `plan`-time gate rather than a manual
review checklist item.

---

## 8. Security

Builds on [security](../04-security.md). Terraform-specific:

**`TF-SEC-01` (MUST)** Never hardcode secrets (API keys, passwords, tokens)
in `.tf` files or `.tfvars` committed to version control — source them from
a secrets manager (Vault, cloud Secret Manager) via a data source, or from
CI-injected environment variables (`GEN-SEC-24`/`25`).

**`TF-SEC-02` (SHOULD)** Default to least-privilege IAM policies scoped to
the specific resources/actions a workload needs — avoid `"*"` actions/
resources in a policy document as a starting template (`GEN-SEC` §1 least
privilege).

**`TF-SEC-03` (SHOULD)** Enable encryption-at-rest and restrict public
network exposure by default for every provisioned data store (require
`tfsec`/Checkov rules for this as a CI gate, not a manual reminder,
§1) — the majority of cloud security-misconfiguration incidents are
default-open storage/network settings (`GEN-SEC` A02:2025).

---

## 9. Testing

Builds on [testing](../05-testing.md). Terraform-specific:

**`TF-TEST-01` (SHOULD)** Use **Terraform's built-in `terraform test`**
framework (native `.tftest.hcl` files) or **Terratest** (Go-based,
provisions real infrastructure in an ephemeral account/project) for module
tests that verify actual provider behavior, not just HCL syntax.

**`TF-TEST-02` (SHOULD)** Validate policy/security constraints with
Checkov/tfsec as an automated test gate in CI (§1), and treat a newly
introduced misconfiguration finding as a test failure, not an informational
warning.

**`TF-TEST-03` (CONSIDER)** Use `terraform plan` against a known-fixture
configuration as a regression test for module behavior — a module change
that unexpectedly alters the plan for an unrelated fixture indicates the
change's blast radius was larger than intended.

---

## 10. Anti-patterns

- Hand-formatted HCL instead of `terraform fmt`; unpinned Terraform/provider
  versions.
- Modules sourced from a mutable ref (`main`/`HEAD`) instead of a pinned
  version.
- `count`-indexed resource sets that shift index on removal, instead of
  `for_each`.
- Untyped/undocumented input variables; secrets not marked `sensitive`.
- Local (non-remote, non-locking) state for anything shared by more than one
  person.
- `.tfstate` committed to version control.
- One monolithic state file for an entire organization.
- `apply` run without reviewing the `plan` diff first.
- Manual hand-editing of `.tfstate` JSON instead of `state mv`/`import`.
- Hardcoded secrets in `.tf`/`.tfvars` files.
- Default-open storage/network configuration with no automated security
  scan gating it.

---

## Quick checklist

- [ ] `terraform fmt` + `terraform validate` clean; tflint + tfsec/Checkov in
      CI; Terraform/provider versions pinned.
- [ ] Resource/variable names don't stutter the type; modules named
      descriptively.
- [ ] Modules extracted at the Rule-of-Three point; narrow single
      responsibility; sources pinned to an exact version.
- [ ] `for_each` used over `count` for addable/removable resource sets.
- [ ] Every variable typed, described, and validated where constrained;
      secret-carrying variables marked `sensitive`.
- [ ] Outputs expose only what consumers need.
- [ ] Remote, locking state backend for shared state; state never committed
      to VCS; encrypted at rest; split by blast radius.
- [ ] `plan` reviewed before every `apply`; destroy/recreate on stateful
      resources requires explicit sign-off.
- [ ] No hardcoded secrets; least-privilege IAM; encryption-at-rest and
      no-public-exposure enforced by automated scan.
- [ ] Module tests (`terraform test`/Terratest); security-policy scan as a
      CI gate, not advisory.

---

## References

- HashiCorp, Terraform documentation — https://developer.hashicorp.com/terraform/docs
- HashiCorp, "Terraform Style Guide" — https://developer.hashicorp.com/terraform/language/style
- HashiCorp, "Standard Module Structure" — https://developer.hashicorp.com/terraform/language/modules/develop/structure
- HashiCorp, "Terraform Cloud/state backends" — https://developer.hashicorp.com/terraform/language/backend
- tflint — https://github.com/terraform-linters/tflint
- tfsec — https://github.com/aquasecurity/tfsec · Checkov — https://www.checkov.io/
- Terratest — https://terratest.gruntwork.io/
- Open Policy Agent / Conftest — https://www.openpolicyagent.org/
