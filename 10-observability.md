# Observability & Logging

> Making production behavior diagnosable: logs, metrics, traces, health checks,
> and structured errors. A system that cannot be observed cannot be operated
> reliably.

**Rule ID prefix:** `GEN-OBS`

---

## Table of contents

1. [Why observability](#1-why-observability)
2. [Logging](#2-logging)
3. [Structured logging](#3-structured-logging)
4. [Log levels](#4-log-levels)
5. [What not to log](#5-what-not-to-log)
6. [Metrics](#6-metrics)
7. [Distributed tracing & correlation](#7-distributed-tracing--correlation)
8. [Health checks & readiness](#8-health-checks--readiness)
9. [Alerting](#9-alerting)
10. [Anti-patterns](#10-anti-patterns)
11. [Quick checklist](#quick-checklist)
12. [References](#references)

---

## 1. Why observability

**`GEN-OBS-01` (SHOULD)** Design for observability from the start. Production
behavior must be diagnosable through logs, metrics, traces, health checks, and
structured errors — not by attaching a debugger to a live system.

**Why:** When an incident happens, you cannot add instrumentation retroactively to
the moment it occurred. The signals you emit *now* are the only forensic record
you will have. Observability shortens incident response, enables performance
analysis, and makes support tractable.

The three classic "pillars": **logs** (discrete events), **metrics** (aggregated
numbers over time), and **traces** (the path of a request across components). They
complement each other; most systems need all three.

---

## 2. Logging

**`GEN-OBS-02` (SHOULD)** Use a real logging framework/library, never `print`/
`console.log`/`Console.WriteLine` for diagnostics. A logger gives you levels,
structured fields, routing to sinks, and runtime configuration that ad-hoc print
statements cannot.

**`GEN-OBS-03` (SHOULD)** Log at **boundaries and important state transitions**
(request received/completed, external call made, job started/finished, state
machine transition, retry/fallback taken) rather than scattering noise on every
line. Each log line should earn its place.

**`GEN-OBS-04` (SHOULD)** Defer message construction so suppressed logs cost
nothing — use the framework's lazy/parameterized form rather than eagerly building
strings (e.g. Python's `logger.info("x=%s", x)`, not an f-string; .NET's message
template `logger.LogInformation("x={X}", x)`). See the language docs for specifics.

---

## 3. Structured logging

**`GEN-OBS-05` (SHOULD)** Prefer **structured logging** (key/value fields, often
JSON) over free-text strings. Use **stable field names** across the codebase
(`user_id`, `order_id`, `duration_ms`, `status`) so logs are queryable and
aggregatable by your log platform.

**Why:** Free-text logs are searchable only by substring. Structured logs let you
filter, group, and chart by field (e.g. "p99 `duration_ms` for `route=/checkout`
where `status>=500`"). Stable field names are what make this possible.

**`GEN-OBS-06` (SHOULD)** Include a **correlation/request ID** (and trace/span IDs
where available) on every log line related to a unit of work, so a single request
can be reconstructed across services and threads (see §7).

---

## 4. Log levels

**`GEN-OBS-07` (SHOULD)** Use levels consistently so operators can tune verbosity:

| Level | Use for |
|-------|---------|
| **TRACE/DEBUG** | Detailed developer diagnostics; off in production by default. |
| **INFO** | Normal, noteworthy events (startup, request summary, job done). |
| **WARN** | Recovered/degraded conditions, retries, fallbacks, deprecations. |
| **ERROR** | Handled failures needing attention; include context + stack. |
| **FATAL/CRITICAL** | Imminent shutdown / unrecoverable condition. |

**`GEN-OBS-08` (SHOULD)** Log a given error **once**, at the level that finally
handles it, with full context — don't log-and-rethrow at every layer
(`GEN-ERR-15`). Duplicate logging hides the real signal.

---

## 5. What not to log

**`GEN-OBS-09` (MUST)** Never log secrets, credentials, API keys, tokens, session
IDs, full payment data, or sensitive PII (`GEN-SEC-39`). Redact or mask sensitive
fields. Logs are widely accessible and frequently a breach vector.

**`GEN-OBS-10` (SHOULD)** Prevent **log injection / forging**: neutralize newlines
and control characters in untrusted data before logging (`GEN-SEC-40`).

**`GEN-OBS-11` (SHOULD)** Watch logging cost and signal-to-noise: excessive logging
increases storage/egress cost, can leak data, and buries the important lines. Log
deliberately, not defensively.

---

## 6. Metrics

**`GEN-OBS-12` (SHOULD)** Emit metrics for **rates, errors, durations, and
saturation** (the "RED"/"USE" families) plus business-critical counters
(orders placed, payments failed). Metrics answer "how often / how fast / how full"
across time, which logs cannot do efficiently.

**`GEN-OBS-13` (SHOULD)** Keep cardinality under control: don't put unbounded
high-cardinality values (user IDs, request IDs, raw URLs) into metric **labels** —
that explodes the metrics backend. Use logs/traces for high-cardinality detail and
metrics for aggregates.

---

## 7. Distributed tracing & correlation

**`GEN-OBS-14` (CONSIDER)** For distributed/microservice systems, propagate a
**trace context** (e.g. W3C Trace Context / OpenTelemetry) across service
boundaries so a request can be followed end-to-end with per-span latency.

**`GEN-OBS-15` (SHOULD)** Generate or accept a correlation ID at the system edge
and thread it through logs, traces, and downstream calls. Without it, multi-service
incidents are nearly impossible to reconstruct.

---

## 8. Health checks & readiness

**`GEN-OBS-16` (SHOULD)** Expose **liveness** (is the process alive?) and
**readiness** (can it serve traffic — dependencies up, warmed?) checks so
orchestrators can restart or route correctly. Don't conflate the two: a service can
be alive but not ready.

**`GEN-OBS-17` (SHOULD)** Make health checks cheap and meaningful — verify critical
dependencies without doing heavy work on every probe.

---

## 9. Alerting

**`GEN-OBS-18` (SHOULD)** Alert on **symptoms users feel** (error rate, latency,
saturation, failed business transactions), not on every internal metric. Each alert
should be actionable and point toward a runbook; noisy alerts train responders to
ignore them.

**`GEN-OBS-19` (SHOULD)** Tie deploys to observability: a deploy without monitoring
and a rollback path is flying blind (`GEN-VCS-21`). Watch the key signals through
and after rollout.

---

## 10. Anti-patterns

- `print`/`console.log` debugging left in production code.
- Free-text-only logs with no structure or stable fields.
- Logging the same error at every layer (duplicate noise).
- Secrets, tokens, or full PII in logs; un-sanitized untrusted input (log
  injection).
- High-cardinality values as metric labels (cardinality explosion).
- No correlation/trace IDs across services.
- Liveness and readiness conflated; heavyweight health checks.
- Alerting on causes/noise instead of user-facing symptoms.
- Deploying with no metrics, traces, or rollback signal.

---

## Quick checklist

- [ ] Real logging framework (no `print`/`console.log`) at boundaries & state
      transitions.
- [ ] Structured logs with stable field names; lazy/deferred message construction.
- [ ] Correlation/request (and trace) IDs on every related log line.
- [ ] Consistent levels; each error logged once with context, not at every layer.
- [ ] No secrets/PII in logs; log injection prevented; logging cost controlled.
- [ ] Metrics for rate/errors/duration/saturation + business counters; bounded
      label cardinality.
- [ ] Trace context propagated across service boundaries (distributed systems).
- [ ] Liveness + readiness checks exposed; cheap and meaningful.
- [ ] Actionable, symptom-based alerts with runbooks.
- [ ] Deploys watched through observability with a rollback path.

---

## References

- Google SRE Book — Monitoring Distributed Systems (the "four golden signals") —
  https://sre.google/sre-book/monitoring-distributed-systems/
- Brendan Gregg, "USE Method" — https://www.brendangregg.com/usemethod.html
- Tom Wilkie, "RED Method" — https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/
- OpenTelemetry — https://opentelemetry.io/docs/
- W3C Trace Context — https://www.w3.org/TR/trace-context/
- The Twelve-Factor App, "Logs" — https://12factor.net/logs
