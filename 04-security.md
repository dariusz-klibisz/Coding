# Secure Coding Practices

> Language-agnostic security guidance. Language-specific security notes live in
> each `languages/` file. This file is grounded primarily in the OWASP Top 10,
> OWASP ASVS, and the SEI CERT secure coding standards.

**Rule ID prefix:** `GEN-SEC`

---

## Table of contents

1. [Security mindset & principles](#1-security-mindset--principles)
2. [OWASP Top 10 mapping](#2-owasp-top-10-2021-mapping)
3. [Input validation & injection](#3-input-validation--injection)
4. [Output encoding](#4-output-encoding)
5. [Authentication](#5-authentication)
6. [Session & access control](#6-session--access-control)
7. [Cryptography](#7-cryptography)
8. [Secrets management](#8-secrets-management)
9. [Memory & resource safety](#9-memory--resource-safety)
10. [Serialization & deserialization](#10-serialization--deserialization)
11. [Dependency & supply-chain security](#11-dependency--supply-chain-security)
12. [Logging, errors & information disclosure](#12-logging-errors--information-disclosure)
13. [Secure defaults & configuration](#13-secure-defaults--configuration)
14. [Anti-patterns](#14-anti-patterns)
15. [Quick checklist](#quick-checklist)
16. [References](#references)

---

## 1. Security mindset & principles

Security is not a feature you add; it is a property of how the whole system is
built. Foundational principles (Saltzer & Schroeder, plus modern practice):

| Principle | Meaning | Why |
|-----------|---------|-----|
| **Least privilege** | Grant the minimum rights needed, for the minimum time. | Limits blast radius of any compromise or bug. |
| **Defense in depth** | Multiple independent layers of control. | No single control is perfect; layers cover each other's gaps. |
| **Fail securely** | On error, default to denied/closed, not open. | A failure must not become an authorization bypass. |
| **Secure by default** | Safe configuration out of the box. | Most users never change defaults. |
| **Complete mediation** | Check authorization on every access, every time. | Cached/assumed permissions are bypass vectors. |
| **Economy of mechanism** | Keep security-critical code small & simple. | Complexity hides flaws; small code is auditable. |
| **Zero trust** | Never trust input, network position, or caller identity implicitly. | Perimeter trust fails; verify explicitly. |
| **Don't roll your own crypto** | Use vetted libraries/protocols. | Crypto is exceptionally easy to get subtly, fatally wrong. |

**`GEN-SEC-01` (MUST)** Treat all data from outside the trust boundary as
hostile until validated (ties to the barricade pattern, `GEN-DEF-16`).

**`GEN-SEC-02` (SHOULD)** Threat-model features that touch authentication,
authorization, money, PII, or external input. Ask: what can an attacker control,
and what's the worst they can do with it?

**Lightweight threat-model process** (do this during design, not after release —
defects are far cheaper to prevent than to patch):

1. **What are we protecting?** (assets: data, funds, availability, integrity)
2. **Who can interact with it?** (entry points, trust boundaries, actors)
3. **What can go wrong?** (spoofing, tampering, disclosure, DoS, escalation)
4. **What controls prevent or detect it?**
5. **How will we verify those controls?** (tests, reviews, monitoring)

---

## 2. OWASP Top 10 (2021) mapping

| # | Category | Primary defenses (this doc) |
|---|----------|------------------------------|
| A01 | Broken Access Control | §6 — enforce authz server-side on every request, deny by default |
| A02 | Cryptographic Failures | §7 — TLS everywhere, strong algorithms, no plaintext secrets |
| A03 | Injection | §3, §4 — parameterized queries, encoding, validation |
| A04 | Insecure Design | §1 — threat modeling, secure design patterns |
| A05 | Security Misconfiguration | §13 — hardened, minimal, secure defaults |
| A06 | Vulnerable/Outdated Components | §11 — dependency scanning, patching |
| A07 | Identification & Auth Failures | §5 — strong auth, MFA, secure session mgmt |
| A08 | Software & Data Integrity Failures | §10, §11 — signed artifacts, safe deserialization |
| A09 | Logging & Monitoring Failures | §12 — log security events, detect anomalies |
| A10 | Server-Side Request Forgery (SSRF) | §3 — validate/allowlist outbound URLs |

---

## 3. Input validation & injection

Injection (SQL, OS command, LDAP, NoSQL, XPath, header, template, log) remains
one of the most damaging vulnerability classes. Root cause: **untrusted data is
interpreted as code/commands.**

**`GEN-SEC-03` (MUST)** Separate code from data. Use parameterized/prepared
statements for SQL; never build queries by string concatenation with user input.

```python
# WRONG — SQL injection
cur.execute(f"SELECT * FROM users WHERE name = '{name}'")

# RIGHT — parameterized
cur.execute("SELECT * FROM users WHERE name = %s", (name,))
```

**`GEN-SEC-04` (MUST)** For OS commands, avoid the shell entirely; pass arguments
as an array to an exec API. Never interpolate untrusted input into a shell string.

```python
# WRONG — command injection
os.system(f"ping {host}")

# RIGHT — no shell, argument vector
subprocess.run(["ping", "-c", "1", host], shell=False, check=True)
```

**`GEN-SEC-05` (MUST)** Validate input at the boundary by type, length, range,
and format, using an **allowlist** (`GEN-DEF-02`–`04`). Reject, don't try to
"sanitize" your way out of malformed input where rejection is possible.

**`GEN-SEC-06` (MUST)** For SSRF protection, validate and allowlist outbound
request destinations; block requests to internal/metadata addresses
(`169.254.169.254`, RFC1918 ranges, `localhost`).

**`GEN-SEC-07` (SHOULD)** Validation is necessary but **not sufficient** against
injection — the authoritative defense is context-correct escaping/parameterization
at the sink (§4). Use both.

**`GEN-SEC-47` (MUST)** Prevent path traversal: any external value used to
construct a filesystem path must be allowlist-validated or reduced to a basename
and resolved against a fixed base directory. The same discipline applies to
interpreter-like sinks that `GEN-SEC-08` does not enumerate: every value
interpolated into a query DSL (e.g. a search-engine filter expression), an
XML/SOAP body, or a URL path segment must be escaped/encoded for that specific
context or validated against a strict allowlist.

```python
# WRONG — "../../etc/passwd" escapes the upload directory
path = os.path.join(UPLOAD_DIR, request.args["filename"])

# RIGHT — reduce to basename, then verify containment after resolution
name = os.path.basename(request.args["filename"])
path = os.path.realpath(os.path.join(UPLOAD_DIR, name))
if not path.startswith(os.path.realpath(UPLOAD_DIR) + os.sep):
    abort(400)
```

---

## 4. Output encoding

**`GEN-SEC-08` (MUST)** Encode/escape data for the specific context it is
rendered into: HTML body, HTML attribute, JavaScript, URL, CSS, SQL, shell, etc.
The correct encoding differs per context.

**Why:** XSS and injection happen when data crosses into an interpreter without
the encoding that neutralizes that interpreter's metacharacters. HTML-encoding
prevents stored/reflected XSS; URL-encoding prevents URL-context attacks. Use the
framework's context-aware auto-escaping (e.g. template engines) and avoid raw
"render as HTML" sinks (`innerHTML`, `v-html`, `dangerouslySetInnerHTML`) with
untrusted data.

**`GEN-SEC-09` (SHOULD)** Set a restrictive Content-Security-Policy and other
security headers (HSTS, `X-Content-Type-Options: nosniff`, `Referrer-Policy`) as
defense-in-depth against XSS and related web attacks.

---

## 5. Authentication

**`GEN-SEC-10` (MUST)** Never store passwords in plaintext or with fast/general
hashes (MD5, SHA-1, raw SHA-256). Use a memory-hard, salted password hash:
**Argon2id** (preferred), **scrypt**, or **bcrypt**.

**Why:** Password hashes will eventually leak; the hash must be expensive to
brute-force. Salting defeats rainbow tables; memory-hardness defeats GPU/ASIC
cracking.

**`GEN-SEC-11` (SHOULD)** Support MFA; enforce strong-password policy aligned with
NIST 800-63B (length over arbitrary complexity, check against breached-password
lists, no forced periodic rotation without cause).

**`GEN-SEC-12` (MUST)** Rate-limit and lock out / back off on repeated failed
authentication to resist credential stuffing and brute force.

**`GEN-SEC-13` (MUST)** Use constant-time comparison for secrets/tokens/MACs to
avoid timing side channels.

---

## 6. Session & access control

**`GEN-SEC-14` (MUST)** Enforce authorization **server-side on every request**,
based on the authenticated identity — never trust client-supplied role/permission
data or hidden form fields (complete mediation).

**`GEN-SEC-15` (MUST)** Deny by default: access is forbidden unless explicitly
granted. Prevents accidental exposure of new endpoints/resources.

**`GEN-SEC-16` (MUST)** Prevent IDOR (Insecure Direct Object Reference): verify
that the current user is authorized for the *specific* object they reference, not
just that they're logged in.

**`GEN-SEC-17` (MUST)** Generate session tokens with a CSPRNG, give them
sufficient entropy, set `HttpOnly`, `Secure`, and `SameSite` cookie attributes,
rotate on privilege change, and expire/invalidate on logout.

**`GEN-SEC-18` (SHOULD)** Protect state-changing requests against CSRF
(anti-CSRF tokens and/or `SameSite` cookies).

---

## 7. Cryptography

**`GEN-SEC-19` (MUST NOT)** Don't invent cryptographic algorithms or protocols.
Use vetted, maintained libraries (libsodium, the platform crypto library) at a
high level (e.g. `crypto_secretbox`, AEAD) rather than assembling primitives.

**`GEN-SEC-20` (MUST)** Use strong, current algorithms: AES-GCM or ChaCha20-
Poly1305 (AEAD) for symmetric encryption; SHA-256/SHA-3 for hashing;
RSA-2048+/ECDSA P-256+/Ed25519 for signatures. Avoid DES/3DES, RC4, MD5, SHA-1,
ECB mode.

**`GEN-SEC-21` (MUST)** Use TLS 1.2+ (prefer 1.3) for all data in transit. No
unencrypted transport for sensitive data.

**`GEN-SEC-22` (MUST)** Source all keys, IVs, nonces, salts, and tokens from a
cryptographically secure RNG (`/dev/urandom`, `secrets`, `crypto.randomBytes`,
`RandomNumberGenerator`) — never `rand()`/`Math.random()`/`random`.

**`GEN-SEC-23` (MUST)** Never reuse a nonce/IV with the same key for AEAD/CTR
modes — it can catastrophically break confidentiality and integrity.

---

## 8. Secrets management

**`GEN-SEC-24` (MUST NOT)** Never hardcode secrets (API keys, passwords, tokens,
private keys) in source code or commit them to version control.

**Why:** Source control history is permanent and widely accessible; leaked
secrets in repos are continuously harvested by automated scanners within minutes
of being pushed.

**`GEN-SEC-25` (SHOULD)** Load secrets from a secrets manager (Vault, cloud KMS/
Secret Manager) or, at minimum, environment variables/secret files outside the
repo. Rotate secrets regularly and on suspected compromise.

**`GEN-SEC-26` (SHOULD)** Run secret-scanning (gitleaks, trufflehog, GitHub
secret scanning) in CI and as a pre-commit hook to catch leaks before they land.

**`GEN-SEC-27` (SHOULD)** Scope secrets minimally (least privilege) and keep them
out of logs, error messages, and crash dumps.

**`GEN-SEC-46` (MUST NOT)** Never transmit secrets (API keys, tokens, signatures,
credentials) in URL query strings. Query strings are captured in server and proxy
access logs, browser history, `Referer` headers, and test traces — none of which
are treated as secret stores. Send secrets in a header (e.g. `Authorization`) or
the request body, and never interpolate a secret-bearing URL into an error
message or log line (`GEN-SEC-27`).

---

## 9. Memory & resource safety

(Primarily for C/C++/unsafe code — see [languages/embedded-c.md](languages/embedded-c.md).)

**`GEN-SEC-28` (MUST)** Prevent buffer overflows: bounds-check all array/buffer
accesses; use length-aware APIs (`snprintf`, `strncpy_s`/`strlcpy`, span types);
never use `gets`, `strcpy`, `sprintf`, `strcat` on untrusted-length data.

**`GEN-SEC-29` (MUST)** Prevent integer overflow/underflow before it feeds size/
index/allocation calculations (a common precursor to heap overflow). Use checked
arithmetic. (SEI CERT INT rules.)

**`GEN-SEC-30` (MUST)** Avoid use-after-free, double-free, and uninitialized
reads. Set pointers to `NULL` after free; prefer RAII/ownership models; use
sanitizers (ASan/UBSan) in CI.

**`GEN-SEC-31` (CONSIDER)** For new systems, prefer memory-safe languages where
feasible; ~70% of severe vulnerabilities in large C/C++ codebases are memory-
safety issues (per Microsoft and Chromium data).

**`GEN-SEC-31a` (SHOULD)** Bound resource consumption to prevent
denial-of-service: cap **request/body sizes**, limit **decompressed output size and
ratio** (defend against zip/decompression bombs), bound array/collection sizes from
untrusted counts, cap recursion/parse depth, and time-box expensive operations.
Unbounded input is a DoS vector even in memory-safe languages.

---

## 10. Serialization & deserialization

**`GEN-SEC-32` (MUST)** Never deserialize untrusted data with formats/libraries
that can instantiate arbitrary types or execute code (Python `pickle`, Java
native serialization, unsafe YAML `load`, .NET `BinaryFormatter`).

**Why:** Insecure deserialization lets an attacker craft a payload that runs code
during deserialization (remote code execution). Prefer data-only formats (JSON,
Protocol Buffers) with strict schemas and explicit type mapping.

**`GEN-SEC-33` (SHOULD)** Verify integrity/authenticity (signatures, MACs) of
serialized data that crosses a trust boundary before processing it.

---

## 11. Dependency & supply-chain security

**`GEN-SEC-34` (MUST)** Keep dependencies updated and continuously scan them for
known vulnerabilities (Dependabot, `npm audit`, `pip-audit`, OWASP Dependency-
Check, Snyk).

**`GEN-SEC-35` (SHOULD)** Pin/lock dependency versions (lockfiles) and verify
integrity hashes for reproducible, tamper-evident builds.

**`GEN-SEC-36` (SHOULD)** Vet new dependencies: maintenance status, popularity,
known CVEs, transitive footprint. Beware typosquatting and dependency-confusion
attacks (prefer scoped/internal registries for private packages).

**`GEN-SEC-37` (CONSIDER)** Generate and retain an SBOM (Software Bill of
Materials); sign build artifacts and verify provenance (SLSA framework).

---

## 12. Logging, errors & information disclosure

**`GEN-SEC-38` (MUST)** Never expose stack traces, internal paths, SQL, or
detailed error internals to end users/clients. Return a generic message + an
error ID; log the detail server-side (ties to `GEN-ERR-15`).

**`GEN-SEC-39` (MUST)** Never log secrets, credentials, full PII, session tokens,
or full card numbers. Mask/redact sensitive fields.

**`GEN-SEC-40` (SHOULD)** Prevent log injection: neutralize newlines/control
characters in untrusted data before logging, so attackers can't forge log
entries.

**`GEN-SEC-41` (SHOULD)** Log security-relevant events (auth success/failure,
access-control denials, input-validation failures, privilege changes) to enable
detection and forensics (A09).

---

## 13. Secure defaults & configuration

**`GEN-SEC-42` (SHOULD)** Ship hardened defaults: least-privilege accounts,
disabled debug/verbose modes in production, no default/sample credentials,
unused features/ports/services disabled.

**`GEN-SEC-43` (SHOULD)** Keep security configuration in code/declarative form
(IaC) and review it; misconfiguration (A05) is among the most common real-world
causes of breaches.

**`GEN-SEC-44` (SHOULD)** Run services as non-root with minimal filesystem and
network permissions; apply container/OS sandboxing.

**`GEN-SEC-45` (MUST)** Fail closed when security configuration is absent. A
missing secret, key, or verification setting must abort startup or reject the
request (e.g. 503) — never silently skip the control. Configuration absence may
disable the *feature*; it must never disable the *control*. The same applies to
shared-secret comparisons where both sides can be absent (an unset env var plus
a missing header compare `undefined === undefined` and "authenticate" nothing),
and to fallback/default secrets or salts baked into code.

**Why:** `if (signingKey) verify(payload)` turns an unset environment variable
into a full authentication bypass — and no test of the correctly-configured path
will catch it. This is the "fail securely" principle (§1) applied to
configuration: the safe response to a missing control is denial, not
permissiveness.

```ts
// WRONG — unset WEBHOOK_SECRET silently disables signature verification
if (process.env.WEBHOOK_SECRET) {
  verifySignature(payload, signature, process.env.WEBHOOK_SECRET);
}
handleWebhook(payload);

// RIGHT — missing secret rejects the request (fail closed)
const secret = process.env.WEBHOOK_SECRET;
if (!secret) throw createError({ statusCode: 503 });        // not configured
if (!verifySignature(payload, signature, secret)) {
  throw createError({ statusCode: 401 });                   // invalid signature
}
handleWebhook(payload);
```

---

## 14. Anti-patterns

- String-concatenated SQL/commands/HTML with untrusted input.
- Client-side-only validation or authorization.
- Rolling your own crypto / using MD5/SHA-1 for passwords.
- Hardcoded or committed secrets.
- `Math.random()`/`rand()` for security tokens.
- Deserializing untrusted data with code-capable deserializers.
- Catch-all that leaks stack traces to users.
- "We'll add security later" — retrofitting is costly and incomplete.
- Trusting `X-Forwarded-For`/headers/JWT claims without verification.
- Ignoring dependency CVE alerts.
- `if (secret) verify(...)` — absent security configuration silently skipping the
  control instead of failing closed.
- Secrets/tokens in URL query strings (logged, cached, leaked via `Referer`).
- Untrusted filenames joined into filesystem paths; unescaped values in query
  DSLs, XML bodies, or URL path segments.

---

## Quick checklist

- [ ] All external input treated as hostile; validated (allowlist) at the boundary.
- [ ] SQL parameterized; OS commands use arg vectors with `shell=false`.
- [ ] External values in file paths basename-reduced/allowlisted with containment
      checks; query-DSL/XML/URL-segment sinks context-escaped or allowlisted.
- [ ] Output context-encoded; no raw-HTML sinks with untrusted data; CSP set.
- [ ] Passwords hashed with Argon2id/scrypt/bcrypt + salt; constant-time compares.
- [ ] Authorization enforced server-side on every request; deny by default; IDOR
      checks present.
- [ ] Session cookies `HttpOnly`/`Secure`/`SameSite`; CSRF protection on mutations.
- [ ] Vetted crypto libs; strong algorithms; TLS 1.2+; CSPRNG for all tokens;
      no nonce reuse.
- [ ] No hardcoded/committed secrets; secret scanning in CI; secrets rotated.
- [ ] No secrets in URL query strings; secrets carried in headers or bodies only.
- [ ] Buffers bounds-checked; integer overflow guarded; no UAF/double-free;
      sanitizers in CI.
- [ ] Resource limits enforced: request/body size, decompression size+ratio,
      collection/recursion depth, operation timeouts (DoS protection).
- [ ] No untrusted deserialization with code-capable formats.
- [ ] Dependencies pinned, scanned, and patched; provenance considered.
- [ ] Errors generic to users, detailed in logs; no secrets/PII in logs; log
      injection prevented; security events logged.
- [ ] Hardened, least-privilege, debug-off production defaults.
- [ ] Missing security configuration fails closed (startup abort or 4xx/5xx),
      never skips the control; no default/fallback secrets or salts in code.

---

## References

- OWASP Top 10 (2021) — https://owasp.org/Top10/
- OWASP Application Security Verification Standard (ASVS) — https://owasp.org/www-project-application-security-verification-standard/
- OWASP Cheat Sheet Series — https://cheatsheetseries.owasp.org/
- OWASP Proactive Controls — https://owasp.org/www-project-proactive-controls/
- SEI CERT C Coding Standard — https://wiki.sei.cmu.edu/confluence/display/c/
- NIST SP 800-63B (Digital Identity / authentication) — https://pages.nist.gov/800-63-3/sp800-63b.html
- Saltzer & Schroeder, "The Protection of Information in Computer Systems."
- SLSA supply-chain framework — https://slsa.dev/
