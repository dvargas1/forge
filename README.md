# Forge

A single Go binary that installs, configures and maintains
[Grafana Alloy](https://grafana.com/docs/alloy/latest/) on a machine, and from
v1.5 centralizes that configuration across a fleet.

> **Status: design phase.** No code yet. The initial architecture decisions are
> locked and recorded in [STATUS.md](STATUS.md). This README describes what Forge
> is being built to do, not what it does today.

## The problem

Alloy is powerful, but it is configured in River, a full configuration language
with components, expressions and cross-references. Managing one host is fine.
Managing twenty means editing files by hand on each one, or writing your own
Ansible roles, and either way nothing tells you which machine diverged from what
you intended.

Forge exists to make Alloy manageable at both ends of that scale: a single host
with a standalone binary, and a fleet with a central server.

## What Forge will do

### Local mode (v1)

- Detect an existing Alloy install, or install and update it from the official
  Grafana package repositories.
- Read the configuration already on the machine, understand what it can, and
  leave the rest untouched.
- Create and maintain integrations from a built-in catalog.
- Validate everything before it touches the running service, and apply with a
  preview, a diff, history and rollback.
- Report service state: version, health, whether the machine's config has
  drifted from what Forge last wrote.

### Server mode (v1.5)

- Run a `remotecfg` listener so machines running Alloy can pull their
  configuration from it.
- Hold the inventory of joined machines, learned from the machines themselves.
- Serve a portal for editing integrations across the fleet.
- Generate a join script that installs and configures Alloy on a new machine and
  points it at the server.

## How it works

**Integrations, not River.** You describe an integration (metrics scrape, log
shipping, OTLP ingestion, host metrics) and Forge validates it and compiles it to
River. Raw River blocks are still allowed as an escape hatch for what the catalog
does not cover.

**Destinations are first class.** An endpoint plus its credentials (a Prometheus
or Mimir, a Loki, an OTLP collector) is its own entity, reused across many
integrations and many machines. It is not duplicated into every integration.

**Adoption, not takeover.** Forge parses existing configuration, adopts the
patterns it recognizes, and preserves everything else verbatim as opaque text. It
never re-prints a block it did not understand, and it backs up the original file
before every write.

**Drift is visible, never silent.** Configuration is hash-checked on every read.
Divergence is shown in the portal with a diff, and adopting it back or re-applying
the intended state is always an explicit choice. Forge does not overwrite on its
own.

**Secrets stay out of the config file.** When a destination needs a credential,
the generated River references a file instead of embedding a literal. The
authoritative value lives only on the server, and the file is provisioned on each
host that needs it.

**Fleet by pull.** Alloy already knows how to fetch its own configuration. A
joined host keeps a minimal local config (the `remotecfg` block and the attributes
that identify it) and pulls everything else from the server, which composes one
full configuration file per machine.

## Roadmap

| Version | Scope |
|---|---|
| v1 | Local mode: install and update Alloy, read existing config, CRUD integrations, validate, preview, apply, rollback. Linux only, systemd. |
| v1.5 | `forge server`: remotecfg listener, fleet inventory, portal, join script. |
| later | Pluggable SQL storage, declarative catalog recipes for third parties, TUI, opt-in drift reconcile, staged rollout. |

## Key decisions

- **Language:** Go, single static binary. Linux only for now, x86_64 and arm64.
- **Surfaces:** CLI plus an embedded web portal. No TUI in v1.
- **Config model:** high-level integrations compiled to River, with a raw-River
  escape hatch.
- **Catalog:** integration types are Go code in a registry, each declaring its
  parameters and its render function. Four types at v1: metrics scrape, logs,
  OTLP receiver, host metrics.
- **State:** on the server, in embedded SQLite behind a storage interface so a
  standalone SQL database can be plugged in later.
- **Auth:** single admin token for the portal, static bearer token over TLS for
  `remotecfg`.

The full list, with rationale, is in [STATUS.md](STATUS.md).

## Build

Nothing to build yet. The repository currently holds the design notes only.

## License

[Apache-2.0](LICENSE). Chosen for compatibility with the `grafana/alloy` syntax
package and the `grafana/alloy-remote-config` protocol definitions that Forge
builds on.

## Not affiliated with Grafana Labs

Forge is an independent project. It is not affiliated with, endorsed by, or
sponsored by Grafana Labs. Grafana and Alloy are trademarks of Grafana Labs and
are used here only to describe interoperability.
