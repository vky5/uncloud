# Cluster state data model

What data actually lives in the cluster store, why each piece is stored, and how it's used. This complements
[`corrosion.md`](./corrosion.md), which covers the sync mechanism itself.

## Two separate databases — don't confuse them

Uncloud actually has **two** SQLite databases per machine, and only one of them is replicated:

| Database | File | Replicated across machines? | Purpose |
|---|---|---|---|
| **Cluster store** | Corrosion's `store.db`, schema in `internal/machine/store/schema.sql` | **Yes**, via Corrosion gossip | Cluster-wide facts every machine needs to agree on: membership, running containers. |
| **Local machine DB** | `machine.db`, schema in `internal/machine/db.go` | **No**, local to the machine only | A local cache of the exact Compose service spec used to create each container on *this* machine, so redeploys can diff old vs. new spec without depending on the cluster store being reachable. |

Everything below the first table is about the **replicated cluster store**, since that's the "Corrosion data model"
in the usual sense. The local `machine.db` is called out separately at the end.

## Table: `cluster` — cluster-wide singleton config

```sql
CREATE TABLE cluster (
    key        TEXT NOT NULL PRIMARY KEY,
    value      ANY,
    updated_at TIMESTAMP NOT NULL DEFAULT '1970-01-01 00:00:00'
);
```

A generic key-value store (`internal/machine/store/store.go`'s `Get`/`Put`/`Delete`) for facts that must be
identical and singular across the whole cluster — the kind of thing you can't let each machine decide independently.
Known keys in use today:

| Key | Value | Why it's stored |
|---|---|---|
| `network` | The mesh-wide CIDR, e.g. `10.210.0.0/16` (`cluster.go:57`) | Every machine's per-machine `/24` subnet is carved out of this range. It's set once at `uc machine init` time and must be the same everywhere, so it lives in the shared store rather than each machine's local config. |
| `created_at` | RFC3339 timestamp of cluster creation (`cluster.go:60`) | Informational — lets any machine report "when was this cluster created" without needing to ask the founding machine specifically. |
| `uncloud_dns` | JSON: `{Endpoint, Name, Token}` describing a reserved managed-DNS domain (`cluster/dns.go:17-25`) | A cluster can reserve exactly one `*.<id>.uncld.dev` domain from the external Uncloud DNS service. The reservation (and its auth token) needs to be visible from any machine so any of them can push DNS record updates — it's inherently cluster-wide, not per-machine. |

This is a generic mechanism (`Get`/`Put`/`Delete` on arbitrary keys), so new singleton settings can be added without
a schema migration — just a new key.

## Table: `machines` — cluster membership directory

```sql
CREATE TABLE machines (
    id         TEXT NOT NULL PRIMARY KEY,
    name       TEXT AS (json_extract(info, '$.name')),
    info       TEXT NOT NULL DEFAULT '{}' CHECK (json_valid(info)),
    created_at TIMESTAMP NOT NULL DEFAULT '1970-01-01 00:00:00',
    updated_at TIMESTAMP NOT NULL DEFAULT '1970-01-01 00:00:00'
);
CREATE INDEX idx_machines_name ON machines (name);
```

One row per machine in the cluster. This **is** the cluster membership list — there's no separate "control plane"
registry; any machine can read this table to know every other machine that exists.

- **`id`** — the machine's stable UUID, assigned once at `machine init`/`machine add` time. Used as the foreign key
  from `containers.machine_id` and as the addressing target for gRPC-proxy forwarding.
- **`name`** — a generated column, extracted from the JSON `info` blob purely so it can be indexed and queried
  efficiently (`ORDER BY name`, uniqueness checks) without parsing JSON on every row.
- **`info`** — the JSON-serialized `pb.MachineInfo` protobuf message (`internal/machine/api/pb/machine.proto`),
  containing:
  - `id`, `name` — duplicated from the row for a self-contained record.
  - `network` (`NetworkConfig`): `subnet` (the machine's `/24`), `management_ip` (its `.1` WireGuard address),
    `endpoints` (reachable IP:port pairs for NAT traversal), `public_key` (WireGuard public key). **This is the
    field that makes zero-config peering possible** — every machine reads every other machine's `network` info from
    this table to configure its own WireGuard peers (`AllowedIPs`, endpoint, public key), with no manual exchange.
  - `public_ip` — used to decide if a machine is internet-reachable (relevant for ingress/Caddy placement and
    managed DNS record updates).
  - `daemon_version`, `docker_version`, `os_pretty_name`, `kernel_version`, `hostname`, `arch` — diagnostic/compat
    info surfaced by `uc machine ls`/`uc machine logs` and used for version-compatibility checks between CLI and
    daemon (`internal/grpcversion`).
- **`created_at` / `updated_at`** — bookkeeping timestamps.

Why store the whole thing as one JSON blob instead of columns per field? It's a protobuf message shared with the
gRPC API (`MachineInfo` is also the wire type returned by `InspectMachine`), so serializing it directly avoids
maintaining a second, parallel schema that has to stay in sync with the `.proto` definition. Only the one field
worth indexing (`name`) gets pulled out as a generated column.

Reads/writes go through `Store.CreateMachine`, `GetMachine`, `ListMachines`, `UpdateMachine`, `DeleteMachine`, and a
live `SubscribeMachines` (`internal/machine/store/store.go:119-334`) that other components use to react instantly to
machines joining, leaving, or updating their info — this is what lets the WireGuard mesh reconfigure itself
automatically when the cluster changes.

## Table: `containers` — running container inventory

```sql
CREATE TABLE containers (
    id           TEXT NOT NULL PRIMARY KEY,
    container    TEXT NOT NULL DEFAULT '{}' CHECK (json_valid(container)),
    machine_id   TEXT NOT NULL DEFAULT '',
    service_id   TEXT AS (json_extract(container, '$.Config.Labels."uncloud.service.id"')),
    service_name TEXT AS (json_extract(container, '$.Config.Labels."uncloud.service.name"')),
    sync_status  TEXT NOT NULL DEFAULT '',
    updated_at   TIMESTAMP NOT NULL DEFAULT '1970-01-01 00:00:00'
);
CREATE INDEX idx_containers_machine_id ON containers (machine_id);
CREATE INDEX idx_containers_service_id ON containers (service_id);
CREATE INDEX idx_containers_service_name ON containers (service_name);
```

One row per Uncloud-managed Docker container, **replicated to every machine**, regardless of which machine actually
runs it. This is the table with the most "moving parts" consumers:

- **`id`** — the Docker container ID.
- **`container`** — JSON-serialized `api.ServiceContainer` (`pkg/api/container.go:178-181`), which is:
  - the full Docker `InspectResponse` (config, state, mounts, network settings, labels — everything `docker inspect`
    would return), embedded directly, **plus**
  - `ServiceSpec` — the Compose-derived spec that produced this container (image, ports, env, volumes, etc.).

  Storing the full Docker inspect payload (not a hand-picked subset) means any consumer — DNS server, Caddy
  controller, `uc ps`/`uc inspect` — can get whatever detail it needs about a container on *any* machine without
  making a live Docker API call to that machine. `service_id`/`service_name` are generated columns pulled out of the
  `uncloud.service.id`/`uncloud.service.name` **labels** buried inside that JSON (`pkg/api/container.go:26-29`), so
  the store can index and filter "all containers for service X" cheaply.
- **`machine_id`** — which machine actually runs this container; foreign key to `machines.id`. This is how the CLI
  and other machines know *where* to route a request that needs the container's local Docker daemon (e.g. `docker
  exec`, log streaming) — via gRPC-proxy forwarding to that specific machine.
- **`sync_status`** — either `"synced"` (the record reflects a fresh read of Docker's actual state,
  `SyncStatusSynced`) or `"outdated"` (the owning machine couldn't confirm the container's state — e.g. it's
  crashing, restarting, or briefly unreachable, `SyncStatusOutdated`). The comment in
  `internal/machine/store/container.go:19-25` is explicit that `"synced"` isn't an absolute guarantee either — you
  also need to check the machine's cluster membership/liveness state to fully trust a record. This exists because,
  in a decentralized system with no single source of truth, a record can only ever be "as fresh as its owning
  machine last confirmed it," and consumers need an explicit signal for that rather than assuming replication delay
  is zero.
- **`updated_at`** — last write time, used to reason about staleness alongside `sync_status`.

**Why this table matters most:** it's the thing that lets DNS resolution, load balancing, and reverse-proxy routing
happen *without a control plane*. The DNS server subscribes to relevant rows and turns them into
`<container>.<machine>.machine.internal` / `<service>.internal` records; Caddy's config controller subscribes to
build its reverse-proxy routes; `uc ls`/`uc ps` query it directly. All of them are reading the same replicated table
— they just query/subscribe to different slices of it.

## Corrosion's own internal tables (not Uncloud's)

Corrosion (via its embedded `cr-sqlite` CRDT extension) maintains its own bookkeeping tables alongside the
application tables above — `crsql_db_versions`, `__corro_bookkeeping`, `__corro_bookkeeping_gaps`. Uncloud doesn't
define these; they're part of how Corrosion implements CRDT versioning and gap detection. Uncloud does read them for
diagnostics:

- `Store.Version` (`store.go:63-88`) reads `crsql_db_versions` (`site_id`, `db_version`) to get a per-machine version
  vector — "what's the latest change version we've seen from each actor."
- `Store.KnownMissingChanges` (`store.go:90-117`) reads `__corro_bookkeeping_gaps` (`actor_id`, `start`, `end`) to
  report ranges of changes a machine knows it's missing from a peer but hasn't received yet — useful for diagnosing
  replication lag or partition recovery.

Neither is part of Uncloud's own schema; they're exposed for observability into Corrosion's replication health.

## The other database: local `machine.db` (not replicated)

```sql
CREATE TABLE IF NOT EXISTS containers (
    id TEXT NOT NULL PRIMARY KEY,
    service_spec TEXT NOT NULL CHECK (json_valid(service_spec)),
    created_at TIMESTAMP NOT NULL DEFAULT (datetime('subsecond')),
    updated_at TIMESTAMP NOT NULL DEFAULT (datetime('subsecond'))
);
```

Defined in `internal/machine/db.go`, this is a **plain local SQLite file** (`/var/lib/uncloud/machine.db`, WAL mode)
— it never touches Corrosion or gossip. Despite the same table name (`containers`), it's a different table serving a
different purpose: for each container running on *this specific machine*, it remembers the exact `service_spec` JSON
that was used to create it (`internal/machine/docker/server.go:752`, read back in `docker/service.go:125`).

Why keep this locally instead of relying on the replicated `container.ServiceSpec` field above? It's a fast, always-
available local lookup the machine's own Docker controller uses when deciding whether a redeploy actually changes
anything — it doesn't need to wait on or trust cluster-wide replication for a decision that's entirely about *this
machine's own containers*. It's a deliberate example of "keep local decisions local," matching Uncloud's broader
imperative-over-declarative philosophy: a machine should be able to reason about its own containers even if it's
currently partitioned from the rest of the cluster.

## Summary: why split state this way

| Concern | Where it lives | Why |
|---|---|---|
| Facts the whole cluster must agree on (mesh CIDR, DNS domain reservation) | `cluster` (replicated) | Singleton, cluster-scoped; every machine needs the same answer. |
| Who's in the cluster and how to reach them | `machines` (replicated) | Needed by every machine to configure WireGuard peers and route requests — this *is* cluster membership. |
| What's running and where | `containers` (replicated) | Needed by every machine to answer DNS/ingress/service-discovery queries without asking the owning machine directly. |
| "Did my redeploy actually change anything for my own containers" | `machine.db` (local, not replicated) | A purely local decision; no reason to pay for replication or depend on cluster reachability. |
