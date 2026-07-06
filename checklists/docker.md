# Docker Review Checklist

> Use with [`../languages/docker.md`](../languages/docker.md) (full rationale and
> rule IDs) and [`general.md`](general.md).

- [ ] hadolint clean; BuildKit features (`--secret`, cache mounts) used where
      applicable (`DOCKER-TOOL-*`).
- [ ] Multi-stage build; stages named (`AS build`), never `--from=0`
      (`DOCKER-IMG-01/02`).
- [ ] Layers ordered by change frequency: manifests → install → source; no
      `COPY . .` before dep install (`DOCKER-IMG-03`).
- [ ] `RUN` combined with cache cleanup in the same layer
      (`--no-install-recommends`, `rm -rf /var/lib/apt/lists/*`) or BuildKit
      cache mounts (`DOCKER-IMG-04`).
- [ ] `.dockerignore` present; excludes `.git`, deps, `.env`, credentials, test
      fixtures (`DOCKER-IMG-05`).
- [ ] Minimal runtime base (alpine/distroless — musl caveat checked); base image
      pinned by tag, by digest in production; `COPY` not `ADD`
      (`DOCKER-IMG-06/07/08`).
- [ ] Non-root `USER` in the final stage (`DOCKER-SEC-01`).
- [ ] No secrets in `ENV`/`ARG` in **any** stage; BuildKit `--secret` for build
      secrets, runtime env for run secrets (`DOCKER-SEC-02`).
- [ ] No `/var/run/docker.sock` mounts; no `privileged: true` — `cap_drop: [ALL]`
      + specific `cap_add` (`DOCKER-SEC-03/04`).
- [ ] Exec-form `CMD`/`ENTRYPOINT` (shell form breaks SIGTERM/graceful shutdown);
      init (`tini`/`--init`) if the app can't be PID 1 (`DOCKER-RUN-01`).
- [ ] `HEALTHCHECK` on long-running services (Kubernetes uses its own probes
      instead) (`DOCKER-RUN-02`).
- [ ] Compose: every `image:` pinned, never `:latest`; digest pins in production
      (`DOCKER-CMP-01/02`).
- [ ] `depends_on` uses `condition: service_healthy` backed by a health check —
      short form only waits for container start (`DOCKER-CMP-03`).
- [ ] Named volumes for persistent data (not bind mounts); profiles for optional
      services (`DOCKER-CMP-04/05`).
- [ ] Env defaults via `${VAR:-default}` (`:-` covers unset **and** empty);
      `deploy.resources` limits on heavy services; no obsolete top-level
      `version:` key (`DOCKER-CMP-06/07/08`).
