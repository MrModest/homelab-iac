# Docker socket proxy options and consumer API needs

Research for [#3](https://github.com/MrModest/homelab-iac/issues/3). Facts and trade-offs only, no decision.
Checked 2026-10-23 against project repos (README, Dockerfile, source on default branch) and official docs.

## 1. Candidates

| | wollomatic/socket-proxy | Tecnativa/docker-socket-proxy | linuxserver/socket-proxy | hectorm/cetusguard | 11notes/socket-proxy |
|---|---|---|---|---|---|
| Latest release | 1.13.1 (2026-08-15) | v0.5.0 (2026-07-27) | 3.4.7-r0-ls103 (2026-10-15) | v1.1.4 (2026-09-03) | v2.2.0 (2026-09-21) |
| Last push / stars | 2026-10-19 / 458 | 2026-08-31 / 2847 | 2026-10-15 / 325 | 2026-10-01 / 93 | 2026-09-21 / 261 |
| Release cadence (last 3) | ~monthly | ~4-7 months apart | weekly-ish (LSIO pipeline rebuilds) | 2026-01, 2026-09 | 2026-02, 2026-09 |
| Engine | Go stdlib `httputil.ReverseProxy`, `FROM scratch` | HAProxy (`haproxy:3.4.2-alpine`) + env-templated cfg | HAProxy from Alpine 3.24 + env-templated cfg | Go, zero deps, `FROM scratch` | Go, distroless |
| Rule granularity | Per HTTP method, Go regex on path (auto-anchored `^…$`), multiple rules per method | Per API section (`CONTAINERS`, `IMAGES`, …) on/off; `POST=1` is a global write switch; a few action toggles (`ALLOW_START/STOP/RESTARTS/PAUSE/UNPAUSE`) | Same sections as Tecnativa, plus `ALLOW_LOGS`, `ALLOW_TOP`, `ALLOW_ARCHIVE`, `ALLOW_EXPORT`, `ALLOW_CHANGES` and libpod sections | Per method list + path regex rules (ABNF grammar, `%API_PREFIX_*%` / `%CONTAINER_ID_OR_NAME%` variables) | Read-only (GET) only, with a fixed blocklist (export, archive, attach/ws, secrets, configs, swarm unlockkey, image get); no allowlist config documented |
| Extra checks | Optional bind-mount source restriction on create endpoints (`-allowbindmountfrom`) | – | – | – | – |
| Per-client rules in one instance | **Yes**: per-container allowlists from Docker labels (`socket-proxy.allow.<method>`), client identified by source IP on a shared network (`-proxycontainername`, since 1.11). Plus `-allowfrom` IP/CIDR/hostname gate for the default list | No: one ruleset per container; distinguish clients by running one instance per consumer on its own network | No: same as Tecnativa | No: one ruleset per instance (multiple `-frontend-addr` share it); optional mTLS client auth, but not per-client rules | No |
| Client access control | `-allowfrom` (default `127.0.0.1/32`), hostnames resolved | None in proxy (network isolation only) | None in proxy | Bind address + optional TLS client certs | None in proxy |
| Listener | TCP `:2375` (default listen `127.0.0.1`) or a unix socket (`-proxysocketendpoint`, mode 0600; disables TCP and per-container labels) | TCP `:2375` | TCP `:2375` (`[::]` v4v6 unless `DISABLE_IPV6=1`) | TCP and/or unix, repeatable `-frontend-addr` | Unix socket `/run/proxy` (owned 1000:1000) + TCP 2375 |
| Image user | `USER 65534:65534`; README example runs `user: "65534:<docker gid>"`, `read_only`, `cap_drop: ALL` | `USER root`; README suggests `--privileged` (SELinux/AppArmor) | root by default (`user root` in haproxy cfg); non-root documented only for rootless Podman; supports `read_only` + tmpfs `/run` | No `USER` in Dockerfile, so root by default (not runtime-tested) | Process starts as root to reach the socket (README); `user: "0:<gid>"` |
| Log streaming (`/containers/{id}/logs?follow=1`) | Works: no server timeouts (`http.Server` without timeouts, explicitly `#nosec G112`); `ReverseProxy` flushes immediately on streaming (Content-Length -1) responses. README's own example client is Dozzle (a log streamer) | Path allowed by `CONTAINERS=1`; default backend has `timeout server 10m` / `timeout client 10m` (inactivity), only `/events` gets `timeout server 0`. A follow stream on a container silent >10 min would be cut by HAProxy (derived from config, not tested) | Needs `CONTAINERS=1` **and** `ALLOW_LOGS=1`; same 10 min inactivity timeouts, and **no** `/events` exemption (events stream also subject to 10 min) (derived from config, not tested) | Works by design: `WriteTimeout: 0 // A positive value closes streaming responses`; handles hijack/upgrade | GET logs not in blocklist, so presumably allowed (not verified in source) |
| Hijack/upgrade (exec, attach) | `httputil.ReverseProxy` supports 101 Upgrade (not tested here) | HAProxy switches to tunnel after 101 (not tested here); `EXEC=1` + `POST=1` needed | Same as Tecnativa | Explicit hijack code path | Blocked (read-only) |

Sources:
[wollomatic README](https://github.com/wollomatic/socket-proxy/blob/main/README.md),
[wollomatic Dockerfile](https://github.com/wollomatic/socket-proxy/blob/main/Dockerfile),
[wollomatic main.go](https://github.com/wollomatic/socket-proxy/blob/main/cmd/socket-proxy/main.go),
[wollomatic handlehttprequest.go](https://github.com/wollomatic/socket-proxy/blob/main/cmd/socket-proxy/handlehttprequest.go),
[Tecnativa README](https://github.com/Tecnativa/docker-socket-proxy/blob/master/README.md),
[Tecnativa haproxy.cfg](https://github.com/Tecnativa/docker-socket-proxy/blob/master/haproxy.cfg),
[Tecnativa Dockerfile](https://github.com/Tecnativa/docker-socket-proxy/blob/master/Dockerfile),
[LSIO README](https://github.com/linuxserver/docker-socket-proxy/blob/main/README.md),
[LSIO haproxy.cfg template](https://github.com/linuxserver/docker-socket-proxy/blob/main/root/templates/haproxy.cfg),
[LSIO Dockerfile](https://github.com/linuxserver/docker-socket-proxy/blob/main/Dockerfile),
[cetusguard README](https://github.com/hectorm/cetusguard/blob/master/README.md),
[cetusguard.go](https://github.com/hectorm/cetusguard/blob/master/cetusguard/cetusguard.go),
[11notes README](https://github.com/11notes/docker-socket-proxy/blob/master/README.md).
Release dates and push dates from the GitHub API.

### Rule-ordering detail (Tecnativa vs LinuxServer)

- **Tecnativa**: the first frontend rule is `http-request deny unless METH_GET || { env(POST) -m bool }`. So `ALLOW_START`/`ALLOW_STOP`/`ALLOW_RESTARTS` only take effect with `POST=1`, and `POST=1` opens every write method on every enabled section (e.g. `CONTAINERS=1` + `POST=1` = create/delete containers).
- **LinuxServer**: the `ALLOW_START/STOP/RESTARTS/PAUSE/UNPAUSE` allow rules come **before** the `POST` deny, so narrow lifecycle writes work with `POST=0`. `ALLOW_LOGS`/`ALLOW_TOP`/`ALLOW_ARCHIVE`/`ALLOW_EXPORT`/`ALLOW_CHANGES` are extra deny gates on sub-paths of `/containers`.
- In HAProxy, `METH_GET` matches `GET` and `HEAD`, so HEAD `/_ping` passes with the defaults (`PING=1`).

Both are derived from reading the shipped config files, not from documentation.

## 2. Consumers

All moby-client-based consumers (Alloy, cAdvisor, Portainer) negotiate the API version on first request: `HEAD /_ping`, falling back to `GET /_ping` ([moby client/ping.go](https://github.com/moby/moby/blob/master/client/ping.go), [client.go](https://github.com/moby/moby/blob/master/client/client.go)). They then call versioned paths (`/v1.xx/...`). wollomatic's README notes Traefik ≥3.6.1 broke without `-allowHEAD=/_ping`; the same applies to any moby client. Regex rules must allow an optional `/v1.xx` prefix.

| Consumer (latest release) | Remote endpoint config | Docker API calls | Writes? | Streaming |
|---|---|---|---|---|
| **Alloy `discovery.docker`** (Alloy v1.21.2, 2026-10-22) | `host` attribute (required, no default); docs example `tcp://localhost:2375`. Does not read `DOCKER_HOST` | `GET /containers/json` (with `filter` blocks), `GET /networks` (for network labels), every `refresh_interval` (default 1m). Network mode `container:X` is resolved from the list, no extra inspect | No | No |
| **Alloy `loki.source.docker`** | `host` attribute; built with `client.WithHost(args.Host)`, http/https schemes get extra HTTP client options | `GET /containers/{id}/json` (inspect, also polled to restart tailers), `GET /containers/{id}/logs?follow=1&stdout=1&stderr=1&timestamps=1&since=…` | No | **Yes**, long-lived follow stream per container; resumes from positions file (`since`) on reconnect |
| **cAdvisor** (v0.60.6, 2026-09-18) | `--docker` flag (default `unix:///var/run/docker.sock`); passed to `WithHost`, so `tcp://` works. Does not read `DOCKER_HOST` | `GET /info`, `GET /version`, `HEAD/GET /_ping`, `GET /images/json`, `GET /containers/{id}/json` | No | No |
| **Glance `docker-containers`** (v0.8.7, 2026-10-15) | `sock-path` property; values starting `tcp://` or `http://` are treated as remote (docs mention docker-socket-proxy explicitly) | Only `GET /containers/json?all=…` (unversioned path, no `_ping`) | No | No |
| **Portainer CE** (2.45.3, 2026-10-21) | `-H/--host` flag ("Specifies the Docker daemon endpoint") or a "Docker Standalone – API" environment with `tcp://host:2375` | Effectively the whole API: container list/inspect/create/start/stop/restart/delete/logs/stats, exec and attach via hijacked `POST /exec/{id}/start` and `POST /containers/{id}/attach` with `Connection: Upgrade, Upgrade: tcp`, images list/pull/build/delete, networks and volumes CRUD, `/info`; compose stacks via embedded Compose v2 library against the same endpoint | **Yes, broad** | Logs + hijacked exec/attach |

### cAdvisor host mounts (does it still need them with a proxy?)

Yes. The Docker API only gives cAdvisor metadata; metrics come from the filesystem:

- `/sys` (ro): cgroup stats, independent of Docker endpoint.
- `/` → `/rootfs` (ro): when not in host namespace, cAdvisor prefixes the Docker root dir with `/rootfs` (`storageDir = path.Join(rootFs, storageDir)` in `container/docker/handler.go`). The Docker root dir comes from `GET /info` (`--docker_root` is a deprecated fallback).
- `/var/lib/docker` (ro): for classic graph drivers (overlay2, zfs, …) cAdvisor reads `image/<driver>/layerdb/mounts/<id>/mount-id` to find the RW layer. If that read fails, the Docker container handler is **not created** (returns error), so per-container Docker metrics are lost.
- `/var/run` (or `/run/containerd`): if `docker info` reports driver `overlayfs` (= containerd snapshotter), cAdvisor opens a containerd client at `--containerd` (default `/run/containerd/containerd.sock`) to load the container spec. That is a **second, unproxied host socket** (containerd's, not Docker's). Docker Engine 29.0 made the containerd image store the default for **fresh installs** ([release notes](https://docs.docker.com/engine/release-notes/29/); not for upgrades or `userns-remap`). Whether the target host uses graph drivers or the containerd store decides whether cAdvisor needs the containerd socket.
- The official run example still mounts `/var/run:/var/run:rw` and `/var/lib/docker:ro`, plus `/dev/disk` for disk metrics ([docs/running.md](https://github.com/google/cadvisor/blob/master/docs/running.md)).

Sources:
[Alloy discovery.docker docs](https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.docker/),
[Alloy loki.source.docker docs](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.docker/),
[Alloy tailer.go](https://github.com/grafana/alloy/blob/main/internal/component/loki/source/docker/tailer.go),
[Alloy docker.go](https://github.com/grafana/alloy/blob/main/internal/component/loki/source/docker/docker.go),
[Prometheus moby/docker.go](https://github.com/prometheus/prometheus/blob/main/discovery/moby/docker.go) (used by `discovery.docker`),
[cAdvisor container/docker](https://github.com/google/cadvisor/tree/master/container/docker),
[Glance configuration.md](https://github.com/glanceapp/glance/blob/main/docs/configuration.md#docker-containers),
[Glance widget-docker-containers.go](https://github.com/glanceapp/glance/blob/main/internal/glance/widget-docker-containers.go),
[Portainer CLI docs](https://docs.portainer.io/advanced/cli),
[Portainer add environment via API](https://docs.portainer.io/admin/environments/add/api),
Portainer `api/http/handler/websocket/exec.go`, `api/ws/hijack.go` (located via code search; operation list via DeepWiki).

## 3. Minimal allowlist per consumer (as facts, not a config proposal)

| Consumer | Methods / paths (optional `/v1.xx` prefix) |
|---|---|
| Alloy (both components) | `HEAD,GET /_ping`; `GET /containers/json`; `GET /networks`; `GET /containers/{id}/json`; `GET /containers/{id}/logs` |
| cAdvisor | `HEAD,GET /_ping`; `GET /version`; `GET /info`; `GET /images/json`; `GET /containers/{id}/json` |
| Glance | `GET /containers/json` |
| Portainer | broad GET/POST/PUT/DELETE incl. exec/attach upgrade; for practical purposes full API |

In section-based proxies (Tecnativa/LSIO) the closest settings are: Alloy `CONTAINERS=1 NETWORKS=1` (+`ALLOW_LOGS=1` on LSIO); cAdvisor `CONTAINERS=1 IMAGES=1 INFO=1`; Glance `CONTAINERS=1`; Portainer needs `POST=1` plus most sections. `CONTAINERS=1` also exposes `/containers/{id}/json` (env vars, incl. secrets passed via env) to all read-only consumers in every option except a regex rule that excludes it (Glance only needs the list).

## 4. Trade-offs observed

- Only **wollomatic** gives different rules to different clients from **one** proxy container (labels + source IP). With the others, per-consumer granularity means one proxy instance per consumer (or per privilege tier), each on its own internal network.
- Only **wollomatic** and **cetusguard** can express per-method, per-path rules; Tecnativa/LSIO grant whole sections.
- **Portainer** needs near-full write access in every option, so its proxy rules give little real restriction over the raw socket; it is the outlier among the foundation consumers.
- **Log streaming**: Go-based proxies (wollomatic, cetusguard) have no idle timeout on streams; HAProxy-based ones (Tecnativa, LSIO) carry a 10 min server/client inactivity timeout on non-events paths. Alloy reconnects with `since`, so a cut stream is recovered, but behaviour was not tested here.
- **Rootless image**: wollomatic ships non-root (65534) and needs the docker group gid; Tecnativa/LSIO/cetusguard/11notes run as root by default.
- **11notes** is read-only and has no configurable allowlist, so it cannot serve Portainer and cannot hide `/containers/{id}/json`.

## 5. Not verified

- No proxy was run; streaming, HEAD `/_ping` and hijack behaviour are read from source/config.
- cetusguard runtime user (only Dockerfile inspected, no `USER` line).
- 11notes: whether `logs` passes and whether any env-based rule configuration exists beyond what the README shows.
- Portainer's exact endpoint list is from DeepWiki plus spot checks of `exec.go`/`hijack.go`, not a full audit; whether Portainer calls `/events` or `/system/df` was not confirmed.
- The LSIO non-root mode on a rootful Docker host (README covers rootless Podman only).
