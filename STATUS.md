# STATUS — Forge

A single Go binary that wraps Grafana Alloy: installs and updates it, reads and
maintains its configuration, and (from v1.5) centralizes the configuration of
many machines through Alloy's native `remotecfg`.

- **License:** Apache-2.0
- **Repo:** `github.com/dvargas1/forge` (public)
- **Language of the project:** English (code, CLI, docs, UI)

## Decisions (initial grilling, 2026-10)

### Shape
- **Stack:** Go, single static binary. No runtime dependencies on the target host.
- **Targets (v1):** Linux only, x86_64 and arm64, systemd.
- **Surfaces:** CLI + embedded web portal (server-rendered templates, no frontend
  build step). TUI explicitly out of v1.
- **Modes:** `forge` (local, standalone) and `forge server` (fleet). Same binary.

### Configuration model
- The user-facing unit is an **integration**: a high-level object (type +
  parameters) that Forge validates and **compiles to River**.
- **Destination** is a separate, reusable entity: endpoint + credentials for a
  Prometheus/Mimir, Loki, or OTLP collector. One destination is reused by many
  integrations across many machines.
- An integration carries a **target expression** (labels/attributes) selecting
  which machines it applies to.
- **Escape hatch:** raw River blocks can be attached when the catalog does not
  cover a case.

### Existing configuration
- Parse with `github.com/grafana/alloy/syntax` (parser / ast / printer).
  Known the API is not stable and the printer is not byte-perfect.
- **Adopt** recognized patterns into the structured model; everything else is
  preserved **verbatim as opaque text** and re-injected untouched.
- **Never reprint a block Forge did not understand.** Backup the original file
  before every write.

### Catalog
- Integration **types are Go code**, registered in a catalog. Each type declares
  its parameter schema and its River render function.
- v1 catalog (4 types):
  1. metrics scrape (`prometheus.exporter.*` + `prometheus.scrape` + `remote_write`)
  2. logs (`loki.source.*` + `loki.write`)
  3. OTLP receiver (`otelcol.receiver.otlp` + exporters)
  4. host metrics (`prometheus.exporter.unix`)

### Secrets
- Least plumbing: a credential exists **only when the destination needs one**.
- When needed, the generated River references a file (`password_file`,
  `bearer_token_file`, `credentials_file`) under `/etc/alloy/secrets/` (0600,
  owned by `alloy`), never an inline literal. `password_file` / `credentials_file`
  are confirmed as supported by Alloy's shared HTTP client config.
- **The master value lives only on the Forge server**, marked **write-only** in the
  portal (Forge writes it and never reads it back).
- **Every host that authenticates needs its own copy of the file**, because Alloy
  reads it locally, on the machine running the pipeline. The server provisions it
  over SSH at join time and on rotation. Unavoidable in any design: the credential
  material must exist on the authenticating host, inline or in a file.
- Why a file beats inline in fleet mode: an inline secret is re-sent over the wire
  and rewritten to disk on every config change; a file is pushed once and rotation
  is decoupled from config rendering (Alloy re-reads the file).
- Rotation: the server rewrites the file on every host using that destination,
  over the same SSH channel used for lifecycle, reporting success or failure per
  host in the portal.

### State
- State lives on the **Forge server**, in embedded **SQLite** behind a storage
  interface, so a standalone SQL server (Postgres) can be plugged in later.
- Local mode keeps its own SQLite on the host.

### Drift
- Config is hash-checked on every read. Divergent config is flagged in the portal
  with a visible diff and **never overwritten automatically**. Adopt-back or
  re-apply is an explicit human choice.

### Auth
- **Portal:** single admin token (env var / flag) + cookie session. Listens on
  localhost by default; TLS required when exposed.
- **remotecfg:** static bearer token over TLS.

### Topology and fleet
- Remote lifecycle operations run over **SSH from the server**. No Forge agent on
  managed hosts: only Alloy runs there.
- The server's inventory of machines **is learned from the remotecfg polls**
  (`RegisterCollector` / `GetConfig` carrying id + attributes), not hand-entered.
- Joined hosts keep a **minimal local config**: the `remotecfg` block plus the
  `attributes` that identify the machine. Everything else is composed server-side
  and delivered as one full config file per machine.
- Bootstrap: the portal generates a **join script** (apt/rpm detection, installs
  Alloy from the official Grafana repo, writes the `remotecfg` block, enables
  systemd) tied to a single-use token with expiry, plus a
  "download and inspect before running" variant.

### Licensing note
Apache-2.0 chosen for compatibility with the imported `grafana/alloy` syntax
package and the `grafana/alloy-remote-config` proto.

## Roadmap
- **v1:** local mode only. Detect / install / update Alloy, read existing config,
  CRUD integrations from the catalog, validate, and apply with preview + diff +
  history + rollback.
- **v1.5:** `forge server`, remotecfg listener, portal, join script.
- **Later:** Postgres storage, declarative catalog recipes for third-party
  contributions, TUI, opt-in drift reconcile, staged rollout.

## Local config on a joined host
A joined host keeps exactly three things locally:

1. the `remotecfg` block (url, token, `attributes`),
2. a `logging` block with a configurable level,
3. an optional `local.alloy` escape hatch for machine-local needs that must keep
   working offline.

Everything else, including the base configuration, is composed server-side and
delivered as one full config file per machine. The portal flags `local.alloy`
content as unmanaged, since it bypasses the server as source of truth. The
`attributes` are the one thing the host must declare itself: they are the key the
server uses to tell machines apart. "Minimal" does not mean "empty".

## Access
- Repo: local `/home/ubuntu/forge`, no git remote configured yet
- Environment: dev on this VPS; no prod deployment

## References
- `/home/ubuntu/alloy-remotecfg-reference.md`
- `/home/ubuntu/alloy-operations-reference.md`
- `/home/ubuntu/alloy-config-management-reference.md`
