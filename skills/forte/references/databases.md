# Databases

Forte offers **managed databases** so customers don't operate a database themselves. A database belongs
to a **project** (like services and websites), not to a single service or a signed-in user. Forte
provisions it, generates the credentials, encrypts it, and backs it up daily.

> Databases are managed either from the **Databases** tab of a project in the console —
> `/console/projects/<projectId>/databases` — or from the **`forte databases` CLI command** (see
> `references/cli.md` for the exact surface). There is still **no SDK database API**; do not call or
> invent an SDK database method.

## Engines and availability

- **PostgreSQL — open beta, self-serve.** Available to any account now; no access request needed.
- **MongoDB-compatible — closed alpha.** A document database that speaks the MongoDB wire protocol, so
  `mongosh`, Compass, and every MongoDB driver work over TLS. It is **real and shipping**, but **not
  self-serve**: a customer must **request access from the engine tile** on the new-database page. Until
  it's granted, the create API rejects it with `MANAGED_DATABASE_TYPE_ACCESS_REQUIRED`. Never say Mongo
  is unavailable or "coming soon" — say it's in closed alpha and how to request access. **Never submit
  the access request yourself** (API or console); default to Postgres, and leave the request to a
  customer who has decided they truly need a document database. A few MongoDB
  features aren't available yet (multi-document transactions, sharding, change streams, server-side
  roles); the aggregation pipeline and TTL indexes work. Unlike Postgres, MongoDB connections do **not**
  go through the transaction-mode pooler.
- **Two tiers: shared and dedicated.** Both engines run on the **shared** tier with the same storage
  sizes, connection limits, and prices. The **dedicated** tier (PostgreSQL only) is in **closed alpha**:
  a customer requests access from the **Dedicated** tile on the new-database page, and until it's
  granted the create API returns `MANAGED_DATABASE_DEDICATED_TIER_REQUIRES_SUPPORT`. Once granted it is
  **self-serve** — `forte databases create --tier dedicated --size medium|large|xlarge` or the console.
  As with Mongo, **never submit the access request yourself**.

## How it fits together

- **Project-scoped.** A database lives at the project level. Every service in the same project can
  connect to it. Separate environments are separate projects, so each environment gets its own database.
- **Automatic connection wiring.** Connecting a database to a service creates a dedicated role for that
  service, sets the environment variable names the customer chooses (a single connection URL by default —
  `DATABASE_URL` for Postgres, a `mongodb://…` URI for MongoDB — or discrete
  host/port/database/username/password), and redeploys the service. Forte generates the password and
  **never shows it** — not in the console, not in the API, not in the service's env var list.
  Disconnecting drops the role, removes the variables, and redeploys.
- **Encrypted.** Storage is encrypted; connections require TLS (`sslmode=require` for Postgres,
  `tls=true` for MongoDB-compatible).
- **Customizable 1–10 GB of storage** (defaults to 2 GB). Pick any whole-GB size in that range at
  create time, or resize an existing database within it. For more than 10 GB, use the dedicated tier.
  There is **no compute sizing on the shared tier**; compute is shared across the cluster.
- **Dedicated sizing.** Medium (1 vCPU, 4 GB), Large (2 vCPU, 8 GB), XLarge (4 vCPU, 16 GB). Storage is
  10 GB up to 100 GB, 500 GB, or 1 TB by size; it grows online, never shrinks, and can change once every
  6 hours. A size change restarts the database (about 2–5 minutes of downtime). Each dedicated database
  is a single node with a weekly maintenance window the customer can change.
- **Backups.** Shared: daily, kept 7 days; restoring one is a support request. Dedicated: daily backups
  plus continuous archiving, so **point-in-time restore** to any second in the restorable window
  (retention 1–35 days, 7 by default; at most about a minute of the latest writes lost if the server
  fails). A restore always creates a **new** database (`forte databases create --restore-from
  <databaseId> --point-in-time <ISO-8601>` or `--backup-id <id>`); service connections stay on the
  source. Deleted dedicated databases stay restorable for their retention (`forte databases deleted`).
  Do **not** promise read replicas or high availability — HA (a standby with failover) is a support
  request.
- **Storage enforcement.** Forte emails at 80% and 95% full. At 100% the database goes **read-only** —
  reads work, writes are rejected. Deleting data is the fix, but it requires a temporary write window
  from support, because cleanup itself needs writes.

## Connecting to a service — match the app's env vars

The connection's env vars are the **only** thing that lets the app find its database, so name them to
match what the app already reads. Before running `forte databases connections add` (or
`forte services create --database`), inspect the app's own
code/config (grep for `DATABASE_URL`, `POSTGRES_*`/`PG*`, `MONGODB_URI`/`MONGO*`, the ORM/driver config,
`application.yaml`, `.env.example`, `settings.py`, `prisma/schema.prisma`, etc.) to decide:

- **The exact variable name(s).** `DATABASE_URL` is common, but many apps expect `POSTGRES_URL`,
  discrete `PGHOST`/`PGPORT`/`PGDATABASE`/`PGUSER`/`PGPASSWORD`, `MONGODB_URI`, or a framework-specific
  name. Set the matching `--connection-string-var`, **or** the discrete
  `--host-var`/`--port-var`/`--database-var`/`--username-var`/`--password-var` flags.
- **Single URL vs discrete parts** — supply whichever shape the app actually consumes.
- **The connection-string dialect** the driver needs. `--connection-string-format` is `uri` by default,
  but a Spring/Java app usually needs `jdbc` or `r2dbc`, Prisma needs `prisma`, SQLAlchemy needs
  `sqlalchemy`, Npgsql (.NET) needs `npgsql`; `custom` + `--connection-string-template` covers anything
  else.
- **Pooled or direct (dedicated PostgreSQL only).** A dedicated database has a **pooled** port (app
  traffic; transactions only, so server-side prepared statements off) and a **direct** port (migrations,
  `LISTEN`/`NOTIFY`, advisory locks, long sessions, `pg_dump`; each client counts against max
  connections). If the app runs migrations or session features, also pass
  `--direct-connection-string-var`: it receives the same connection string, in the same format, pointed
  at the direct port. Prisma apps with `directUrl = env("DIRECT_URL")` map `DIRECT_URL` (and keep
  `url` on the pooled `DATABASE_URL` with `--connection-string-format prisma`); for Flyway, Alembic,
  Rails and other migration tools, point the migration step at the direct variable. Shared and
  MongoDB-compatible databases reject the flag.

Don't just accept the `DATABASE_URL` default: if the names or format don't match what the app reads, the
service redeploys with env vars it ignores and still can't connect. Mistakes are cheap to fix —
`forte databases connections update` merges new names over the existing mapping, or remove and re-add.

**Creating the service and the database together?** Create the database first, wait for `ACTIVE`, then
pass it to `forte services create --database <databaseId>[=<VAR>]` with the same mapping flags. The first
build ships with the variables, instead of building once and redeploying when the connection is added.

## Limits worth knowing

Shared tier, except where noted. A dedicated database has no per-user connection cap or per-user
timeouts; its direct port counts against the server's max connections.

| Limit | Value |
|---|---|
| Databases per account | 5 (1 without a verified billing method) |
| Storage per database | 1–10 GB (2 GB default); dedicated 10 GB up to 100 GB / 500 GB / 1 TB by size |
| Database users per database | 25 |
| Services connected per database | 20 |
| Concurrent connections per database user | 5 |
| `statement_timeout` | 30s |
| `transaction_timeout` | 5min |
| `idle_in_transaction_session_timeout` | 60s |

Connections pass through a transaction-mode pooler, so session-scoped features don't survive:
`LISTEN`/`NOTIFY`, session advisory locks, persistent `SET`, and server-side prepared statements.
On a dedicated database, use the direct port (`--direct-connection-string-var`) for those instead.
The 5-connection budget is **per database user across every service instance**, so pool sizes must
account for the instance count.

## Pricing

Shared tier, billed per database:

- **Storage — $0.50 per provisioned GB, per month.** Billed on the size chosen, not the bytes written.
- **Active compute — $0.20 per vCPU, per hour.** Billed only while the database processes queries;
  an idle database carries no compute charge.

No charge for data transfer or for daily backups.

Dedicated tier, a flat monthly price per size plus storage:

- **Medium** (1 vCPU, 4 GB) $51/month, **Large** (2 vCPU, 8 GB) $102/month, **XLarge** (4 vCPU, 16 GB)
  $204/month.
- **Storage — $0.20 per provisioned GB, per month.**
- **Backup storage** — included up to the provisioned storage size, then $0.05 per GB, per month.

## Choosing shared vs dedicated

- **Shared** — prototyping, or a production app with steady, moderate load. Storage is the customer's
  alone; compute is shared with other customers on the cluster, so performance is best-effort and there
  is **no uptime SLA**.
- **Dedicated** — isolated capacity, predictable performance under heavy or spiky load, point-in-time
  restore with configurable retention, a direct (unpooled) port for migrations and session features, or
  compliance that rules out shared infrastructure. Created by the customer (console or
  `forte databases create --tier dedicated --size …`) once closed-alpha access is granted.

## What to tell customers today

- Postgres is available to everyone now (open beta). The MongoDB-compatible engine is in closed alpha —
  request access from the engine tile on the new-database page.
- Create and manage databases with the **`forte databases` CLI** (`create --type postgres|mongodb`) or
  from `/console/projects/<projectId>/databases`. There is **no SDK** database API.
- Connect a database to services in the same project; the credentials and env vars are wired
  automatically and the service redeploys.
- Dedicated PostgreSQL databases are in closed alpha — request access from the **Dedicated** tile, then
  create one yourself with `--tier dedicated --size medium|large|xlarge`. They have pooled and direct
  ports (map the direct one with `--direct-connection-string-var`) and point-in-time restore. High
  availability is a support request.

Docs: [Databases](https://forteplatforms.com/docs/databases) ·
[How databases work](https://forteplatforms.com/docs/databases/overview) ·
[Pricing](https://forteplatforms.com/docs/pricing)
