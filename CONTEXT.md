# Homelab

Declarative description of a single home server: the host it runs on and the app stacks it serves. Reading the declarations should tell the story of the homelab without reading task code.

## Declarations

**Homelab declaration**:
The one file (`homelab.yml`) stating what the homelab is: domain, LAN address book, pools, locations, networks, house rules, external services. Covers the apps server only.
_Avoid_: global vars, group_vars, inventory

**Stack**:
One app as a unit of deployment: its containers, data locations, exposure and metrics, declared in `stacks/<slug>/stack.yml`.
_Avoid_: app, service, role

**Slug**:
A stack's identifier; names its compose project, data folders and container-name prefix.

**Container**:
One entry in a stack's `containers` map; becomes one compose service.
_Avoid_: service (overloaded with Compose and systemd)

**House rule**:
A default declared once in the homelab declaration (runtime user, naming, restart policy, storage mode) that every stack inherits.
_Avoid_: convention, global default

**Departure**:
A stack-level override of a house rule. Always carries a `reason`.
_Avoid_: exception, override

**Settings**:
Stack-scoped named values shared by env, commands and content templates.

## Access

**Admin user**:
The one account that logs in over SSH, used by both the operator and Ansible.
_Avoid_: ssh user, ansible user

**Recovery user**:
A console-only account with its own password, kept for physical access when SSH fails. Never reachable over SSH.

**Runtime user**:
The account (`apps`) that containers run as and that owns stack data by default.
_Avoid_: docker user, service user

## Storage

**Pool**:
A ZFS pool on the host with a declared purpose (e.g. `fast`, `slow`).

**Location**:
A named place where stack data lands (`fast`, `slow`, `backups`, `media`), resolved to a path on a pool.
_Avoid_: volume, dir

**Host mount**:
A mount of an existing host path (`host:`); never created or re-owned.

**Content**:
Files Ansible owns inside a location, rewritten on every deploy.

**Seed**:
Files written into a location only when absent; the app owns them afterwards.

## Index and consumers

**Services index**:
The committed, generated file (`generated/services.yml`) holding routes, scrape targets, dashboard tiles, backups and the stack/image view. Built from declarations only, never from running state.

**Consumer**:
A shared service configured only from the services index (reverse proxy, metrics collector, dashboard).

**Route**:
One host name on the reverse proxy forwarding to a container or external service, derived from an `expose` entry.

**Egress**:
A container's declared need to reach beyond the host (internet or LAN), with a reason. Without it, a container reaches only its own stack and the consumers wired to it.

**External service**:
Something reachable through the reverse proxy but not deployed as a stack (e.g. Pi-hole on the Pi).

## Lifecycle

**Parked stack**:
A stack with `enabled: false` and a reason: brought down, data kept, removed from routes, targets and tiles.
_Avoid_: disabled, removed

**Backup agent**:
A per-database sidecar container, using the database's own image, that dumps that one database on a schedule.
_Avoid_: backup job, dump cron
