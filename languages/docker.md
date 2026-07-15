# Docker (`Dockerfile`, Compose) — Coding Standards & State-of-the-Art Practices

> **Scope:** Dockerfiles (BuildKit-era syntax) and Docker Compose v2 files under
> the Compose Specification. Covers image builds, container security, runtime
> behavior (signals, health checks), and Compose service definitions. Generic YAML
> style for Compose files lives in [yaml.md](yaml.md); this file covers the
> Docker-specific semantics.
> **Primary sources:** Docker docs "Building best practices", Dockerfile
> reference, Compose Specification.
> **Relationship to general docs:** extends [general docs](../00-index.md) —
> especially [security](../04-security.md) and
> [tooling & automation](../11-tooling-and-automation.md). Where a general rule
> and a Docker rule conflict, the Docker rule wins for Docker files.

**Rule ID prefix:** `DOCKER`

---

## Table of contents

1. [Tooling & enforcement](#1-tooling--enforcement)
2. [Image builds & layers](#2-image-builds--layers)
3. [Security](#3-security)
4. [Runtime behavior: PID 1, signals, health checks](#4-runtime-behavior-pid-1-signals-health-checks)
5. [Docker Compose](#5-docker-compose)
6. [Anti-patterns](#6-anti-patterns)
7. [Quick checklist](#quick-checklist)
8. [References](#references)

---

## 1. Tooling & enforcement

**`DOCKER-TOOL-01` (SHOULD)** Lint every Dockerfile with **hadolint** in CI. It
encodes most of the rules below (shell-form `CMD`, unpinned versions, missing
`--no-install-recommends`, `apt` cache cleanup) as automated checks, and pipes
`RUN` lines through ShellCheck.

**`DOCKER-TOOL-02` (SHOULD)** Build with **BuildKit** (the default builder in
current Docker releases). BuildKit provides secret mounts (`--mount=type=secret`),
cache mounts (`--mount=type=cache`), and parallel stage execution that several
rules below depend on. Declare the syntax you need where relevant
(`# syntax=docker/dockerfile:1`).

---

## 2. Image builds & layers

**`DOCKER-IMG-01` (MUST)** Use **multi-stage builds**: a build stage with
compilers, dev dependencies, and build tooling, and a separate minimal runtime
stage that copies in only the built artifacts.

**Why:** the runtime image ships no compilers, package caches, or dev
dependencies — smaller images, faster pulls, and a much smaller attack surface.
Build-time files never enter the final image's layers.

```dockerfile
# Bad — single stage: dev deps and build tooling ship to production
FROM node:22
COPY . .
RUN npm ci && npm run build
CMD ["node", "dist/server.js"]

# Good — build stage discarded; runtime gets artifacts only
FROM node:22 AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS runtime
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
USER node
CMD ["node", "dist/server.js"]
```

**`DOCKER-IMG-02` (MUST)** Name build stages (`FROM … AS build`) and reference
them by name (`COPY --from=build`) — never by numeric index (`--from=0`).
Numeric references silently break when stages are added or reordered; names
survive reordering and document intent.

**`DOCKER-IMG-03` (SHOULD)** Order layers by **change frequency**: copy dependency
manifests first (`package.json` + lockfile, `requirements.txt`,
`go.mod`/`go.sum`), run the install, then copy the source tree.

**Why:** each instruction's cache is invalidated when its inputs change, and
invalidation cascades to all later layers. `COPY . .` before the install means
every source edit re-runs dependency installation; manifests-first means the
dependency layer is reused on every rebuild that only touches source.

```dockerfile
# Bad — any source change busts the dependency cache
COPY . .
RUN npm ci

# Good — dependency layer cached until the manifests change
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

**`DOCKER-IMG-04` (SHOULD)** Combine related `RUN` commands and clean package-
manager caches **in the same layer**; use `--no-install-recommends` for apt.

**Why:** each `RUN` creates a layer. A cleanup in a *later* layer does not shrink
the image — the files still exist in the earlier layer. A separate
`apt-get update` layer also caches a stale package index, so later installs can
fetch outdated or vanished packages.

```dockerfile
# Bad — three layers; cache deleted in a layer of its own (no size win)
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# Good — one layer: update, minimal install, cleanup together
RUN apt-get update && apt-get install -y --no-install-recommends curl \
  && rm -rf /var/lib/apt/lists/*
```

> **Note (modern alternative):** with BuildKit, a cache mount keeps the package
> cache out of the image *and* reuses it across builds:
> `RUN --mount=type=cache,target=/var/cache/apt apt-get update && apt-get install -y …`.
> The same pattern works for npm/pip/Go module caches.

**`DOCKER-IMG-05` (MUST)** Maintain a `.dockerignore` that excludes `.git`,
`node_modules` (or equivalent), local `.env` files, credentials, test fixtures,
and docs from the build context.

**Why:** two distinct benefits. First, a smaller build context uploads faster and
avoids needless cache invalidation. Second, it is a **secret-leak barrier**: a
broad `COPY . .` copies everything in the context — without `.dockerignore`, a
local `.env` or key file silently becomes part of an image layer.

**`DOCKER-IMG-06` (SHOULD)** Use minimal base images for runtime stages — Alpine
or distroless. They reduce image size and attack surface (fewer packages, fewer
CVEs to patch; distroless has no shell at all).

**Pros:** small pulls, minimal CVE surface, faster cold starts.
**Cons / when to avoid:** Alpine uses **musl** instead of glibc — native modules
and prebuilt binaries compiled against glibc may fail or need recompilation, and
some DNS/locale behavior differs. Distroless images have no shell or package
manager, which complicates debugging (use ephemeral debug containers). If native
dependencies fight you, a `-slim` glibc image is a reasonable middle ground.

**`DOCKER-IMG-07` (MUST)** Pin base images to a specific version tag — never
`FROM node:latest`. In production, **pin by digest**
(`FROM node:22-alpine@sha256:…`): tags are mutable and can be re-pushed to point
at different content; digests are immutable and make builds reproducible.

**`DOCKER-IMG-08` (SHOULD)** Use `COPY` instead of `ADD` for local files. `ADD`
has magic behaviors — it auto-extracts local tar archives and (historically)
fetches URLs — that make it unpredictable. Reserve `ADD` for the rare case where
tar auto-extraction is exactly what you want; use `COPY` everywhere else.

---

## 3. Security

Builds on [security](../04-security.md).

**`DOCKER-SEC-01` (MUST)** Run as non-root: create a dedicated user and switch to
it with `USER` in the final stage, before `CMD`/`ENTRYPOINT`. A root process
inside a container is root on the host kernel; combined with a kernel or runtime
vulnerability this enables container escape. Many official images ship a ready
non-root user (e.g. `node`, `nobody` in distroless `:nonroot` variants).

```dockerfile
FROM alpine:3.20 AS runtime
RUN addgroup -S app && adduser -S app -G app
USER app
CMD ["/app/server"]
```

**`DOCKER-SEC-02` (MUST NOT)** Never pass secrets as `ENV` or `ARG` in **any**
stage — not even a discarded build stage.

**Why:** `ENV` values persist in the image configuration and are visible to
anyone with the image via `docker inspect` and `docker history`. `ARG` values
are recorded in the layer history and the build cache of every stage that uses
them. "It was only in the build stage" does not make it safe. Use BuildKit
secret mounts for build-time secrets and runtime environment variables (or a
secret store / vault sidecar) for run-time secrets.

```dockerfile
# Bad — token recorded in image config / layer history
ARG NPM_TOKEN
ENV NPM_TOKEN=${NPM_TOKEN}
RUN npm ci

# Good — secret mounted only for the duration of the RUN, never in a layer
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
# build with: docker build --secret id=npmrc,src=$HOME/.npmrc .
```

**`DOCKER-SEC-03` (MUST NOT)** Never mount `/var/run/docker.sock` into a
container. Access to the Docker socket is full control of the Docker daemon —
equivalent to root on the host (the container can start privileged containers,
mount the host filesystem, etc.). If a workload genuinely needs container
management, use rootless Docker, Podman, or a filtering socket proxy that
exposes only the required API calls.

**`DOCKER-SEC-04` (SHOULD NOT)** Avoid `privileged: true` — it grants all
capabilities and lifts device/cgroup restrictions. Grant the specific
capabilities a service needs with `cap_add` (ideally after `cap_drop: [ALL]`):

```yaml
services:
  vpn:
    cap_drop: [ALL]
    cap_add: [NET_ADMIN]   # the one capability actually required
```

---

## 4. Runtime behavior: PID 1, signals, health checks

**`DOCKER-RUN-01` (MUST)** Use the **exec form** of `CMD` and `ENTRYPOINT`
(`CMD ["node", "server.js"]`), not the shell form (`CMD node server.js`).

**Why:** shell form wraps the command in `/bin/sh -c`, so the *shell* becomes
PID 1 and the application a child process. The shell does not forward signals,
so `SIGTERM` from `docker stop` never reaches the app — no graceful shutdown
(connections dropped, buffers unflushed), and the container is `SIGKILL`ed after
the stop timeout.

```dockerfile
# Bad — sh is PID 1; SIGTERM never reaches node
CMD node server.js

# Good — node is PID 1 and receives SIGTERM directly
CMD ["node", "server.js"]
```

> **Note:** exec form does no variable expansion and the app must handle
> `SIGTERM` itself. If the process cannot (or you need zombie reaping), use a
> minimal init as entrypoint (`docker run --init`, or `tini`).

**`DOCKER-RUN-02` (SHOULD)** Define a `HEALTHCHECK` on every long-running
service image. Compose consumes it for startup ordering
(`depends_on: condition: service_healthy`, see `DOCKER-CMP-03`), and
orchestration/monitoring can distinguish "process running" from "service
actually able to do work".

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD ["curl", "-f", "http://localhost:8080/health"]
```

> **Note:** **Kubernetes ignores the Dockerfile `HEALTHCHECK`** — it uses its
> own liveness/readiness/startup probes. Define probes in the pod spec when
> deploying to Kubernetes; keep the Dockerfile `HEALTHCHECK` for Compose and
> plain Docker.

---

## 5. Docker Compose

Compose v2 under the **Compose Specification**. General YAML style
(indentation, quoting, anchors) is covered in [yaml.md](yaml.md).

**`DOCKER-CMP-01` (MUST)** Pin every `image:` to a specific version tag — never
`image: postgres:latest` (or a bare `image: postgres`, which implies `:latest`).
`latest` is non-reproducible: each host pulls whatever the tag points at that
day, so environments silently diverge and upgrades happen by accident.

**`DOCKER-CMP-02` (SHOULD)** In production, pin images by **digest**:

```yaml
image: postgres:16-alpine@sha256:def456…
```

Tags are mutable (a `16-alpine` re-push changes what you deploy); digests are
content-addressed and immutable.

**`DOCKER-CMP-03` (MUST)** Use `depends_on` with
`condition: service_healthy` (backed by a health check, `DOCKER-RUN-02` or a
Compose-level `healthcheck:`) for startup ordering. The short form
(`depends_on: [db]`) only waits for the container to **start**, not for the
service inside it to be ready — the classic "app crashes because Postgres is
still initializing" race.

```yaml
services:
  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 10
  api:
    depends_on:
      db:
        condition: service_healthy
```

> **Note:** the old advice that "Compose file format v3 removed
> `depends_on.condition`" is **obsolete**. The unified Compose Specification
> (Compose v2) supports `condition: service_healthy` / `service_started` /
> `service_completed_successfully`.

**`DOCKER-CMP-04` (SHOULD)** Use **named volumes** for persistent data
(databases, indexes, uploads), not bind-mounted host paths. Named volumes are
portable across hosts and CI, managed by Docker (permissions, lifecycle), and
don't depend on host directory layout. Reserve bind mounts for development
source mounting and for injecting config files.

**`DOCKER-CMP-05` (SHOULD)** Gate optional services behind Compose
**profiles** (`profiles: ["analytics"]`) and enable them via
`COMPOSE_PROFILES` / `--profile`. The default `up` then starts only the core
stack; optional subsystems are opt-in without maintaining parallel Compose
files.

**`DOCKER-CMP-06` (SHOULD)** Give interpolated environment variables defaults
with `${VAR:-default}` so the stack behaves sensibly when a variable is absent
from `.env`:

```yaml
environment:
  - LOG_LEVEL=${LOG_LEVEL:-info}
```

> **Note:** `${VAR:-default}` applies the default when the variable is **unset
> or empty**; `${VAR-default}` only when it is **unset**. Prefer `:-` — an
> accidental empty assignment in `.env` (`LOG_LEVEL=`) otherwise injects an
> empty string.

**`DOCKER-CMP-07` (SHOULD)** Set `deploy.resources` limits (memory, CPU) on
heavy or optional services so one misbehaving container cannot starve the host:

```yaml
deploy:
  resources:
    limits:
      memory: 512M
      cpus: "1.0"
```

> **Note:** under Compose v2 these limits are honored by plain
> `docker compose up` — the old "`deploy:` requires Swarm" caveat no longer
> applies to resource limits.

**`DOCKER-CMP-08` (SHOULD NOT)** Don't add the top-level `version:` key to new
Compose files. It is **obsolete** under the Compose Specification — Compose v2
ignores it (emitting a warning) and always applies the latest specification.

---

## 6. Anti-patterns

| Anti-pattern | Why harmful | Fix |
|---|---|---|
| `image: latest` / unpinned `FROM` | Non-reproducible; breaks silently | Pin to exact version tag; digest in production (`DOCKER-IMG-07`, `DOCKER-CMP-01/02`) |
| `privileged: true` | Full host capabilities; security risk | Drop all, add specific capabilities with `cap_add` (`DOCKER-SEC-04`) |
| Secrets in `ENV`/`ARG` (any stage) | Visible in `docker inspect`/`docker history`; cached | BuildKit `--secret` mount or runtime env (`DOCKER-SEC-02`) |
| Numeric stage references (`--from=0`) | Breaks on reorder | Named stages (`AS build`, `AS runtime`) (`DOCKER-IMG-02`) |
| Root process (no `USER`) | Container escape risk | `USER app` in the final stage (`DOCKER-SEC-01`) |
| Separate `apt-get update` layer | Layer caches stale package index | Combine with install in one `RUN` (`DOCKER-IMG-04`) |
| `COPY . .` before dep install | Every source change busts the deps cache | Copy manifests first, install, then copy source (`DOCKER-IMG-03`) |
| Shell form `CMD` | Shell is PID 1; SIGTERM not forwarded | Exec form `CMD ["node", "…"]` (`DOCKER-RUN-01`) |
| Mounting `/var/run/docker.sock` | Container gains full Docker daemon control (host root) | Rootless Docker; Podman; filtering socket proxy (`DOCKER-SEC-03`) |
| `ADD` instead of `COPY` | Auto-extracts tarballs; unpredictable | `COPY` for all local files (`DOCKER-IMG-08`) |
| Missing `.dockerignore` | Slow context; `.env`/keys leak into layers via `COPY . .` | Ignore `.git`, deps, `.env`, credentials (`DOCKER-IMG-05`) |
| Short-form `depends_on` as readiness | Only waits for container start | `condition: service_healthy` + health check (`DOCKER-CMP-03`) |
| Bind mounts for database data | Host-dependent; breaks in CI; permission drift | Named volumes (`DOCKER-CMP-04`) |
| Top-level `version:` in new Compose files | Obsolete; ignored with a warning | Omit it (`DOCKER-CMP-08`) |

---

## Quick checklist

- [ ] hadolint in CI; BuildKit builds (`DOCKER-TOOL-*`).
- [ ] Multi-stage build; named stages, no `--from=0` (`DOCKER-IMG-01/02`).
- [ ] Layers ordered manifests → install → source; `RUN` combined with cache
      cleanup in the same layer (`--no-install-recommends`,
      `rm -rf /var/lib/apt/lists/*`, or BuildKit cache mounts)
      (`DOCKER-IMG-03/04`).
- [ ] `.dockerignore` excludes `.git`, deps, `.env`, credentials
      (`DOCKER-IMG-05`).
- [ ] Minimal base image (alpine/distroless; musl caveat checked); base pinned by
      tag, by digest in production (`DOCKER-IMG-06/07`).
- [ ] `COPY`, not `ADD`, for local files (`DOCKER-IMG-08`).
- [ ] Non-root `USER` in the final stage (`DOCKER-SEC-01`).
- [ ] No secrets in `ENV`/`ARG` in any stage; BuildKit `--secret` or runtime env
      (`DOCKER-SEC-02`).
- [ ] No `/var/run/docker.sock` mounts; no `privileged: true` — specific
      `cap_add` instead (`DOCKER-SEC-03/04`).
- [ ] Exec-form `CMD`/`ENTRYPOINT`; app (or an init) handles SIGTERM
      (`DOCKER-RUN-01`).
- [ ] `HEALTHCHECK` on long-running services (K8s uses its own probes)
      (`DOCKER-RUN-02`).
- [ ] Compose: images pinned (never `:latest`), digests in prod
      (`DOCKER-CMP-01/02`).
- [ ] `depends_on` uses `condition: service_healthy` (`DOCKER-CMP-03`).
- [ ] Named volumes for persistent data; profiles for optional services
      (`DOCKER-CMP-04/05`).
- [ ] Env defaults via `${VAR:-default}`; `deploy.resources` limits on heavy
      services; no obsolete `version:` key (`DOCKER-CMP-06/07/08`).

---

## References

- Docker docs — Building best practices —
  https://docs.docker.com/build/building/best-practices/
- Dockerfile reference — https://docs.docker.com/reference/dockerfile/
- Build secrets (BuildKit `--mount=type=secret`) —
  https://docs.docker.com/build/building/secrets/
- Compose Specification — https://docs.docker.com/reference/compose-file/
  (canonical spec: https://github.com/compose-spec/compose-spec)
- Compose profiles — https://docs.docker.com/compose/how-tos/profiles/
- hadolint (Dockerfile linter) — https://github.com/hadolint/hadolint
