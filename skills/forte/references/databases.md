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
  is unavailable or "coming soon" — say it's in closed alpha and how to request access. A few MongoDB
  features aren't available yet (multi-document transactions, sharding, change streams, server-side
  roles); the aggregation pipeline and TTL indexes work. Unlike Postgres, MongoDB connections do **not**
  go through the transaction-mode pooler.
- **Shared tier only, self-serve.** The **dedicated** tier is **contact support** — it is not something
  a customer can create in the console. Both engines run on the shared tier with the same storage sizes,
  connection limits, and prices.

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
  create time, or resize an existing database within it. Larger than 10 GB is **contact support**,
  either as a per-account exception or the dedicated tier. There is **no compute sizing on the shared
  tier**; compute is shared across the cluster.
- **Daily backups.** Restore is a support request. Do **not** promise point-in-time recovery, read
  replicas, high availability, or a recovery-point objective.
- **Storage enforcement.** Forte emails at 80% and 95% full. At 100% the database goes **read-only** —
  reads work, writes are rejected. Deleting data is the fix, but it requires a temporary write window
  from support, because cleanup itself needs writes.

## Connecting to a service — match the app's env vars

The connection's env vars are the **only** thing that lets the app find its database, so name them to
match what the app already reads. Before running `forte databases connections add`, inspect the app's own
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

Don't just accept the `DATABASE_URL` default: if the names or format don't match what the app reads, the
service redeploys with env vars it ignores and still can't connect. Mistakes are cheap to fix —
`forte databases connections update` merges new names over the existing mapping, or remove and re-add.

## Limits worth knowing

| Limit | Value |
|---|---|
| Databases per account | 5 (1 without a verified billing method) |
| Storage per database | 1–10 GB (2 GB default) |
| Database users per database | 25 |
| Services connected per database | 20 |
| Concurrent connections per database user | 5 |
| `statement_timeout` | 30s |
| `transaction_timeout` | 5min |
| `idle_in_transaction_session_timeout` | 60s |

Connections pass through a transaction-mode pooler, so session-scoped features don't survive:
`LISTEN`/`NOTIFY`, session advisory locks, persistent `SET`, and server-side prepared statements.
The 5-connection budget is **per database user across every service instance**, so pool sizes must
account for the instance count.

## Pricing

Shared tier, billed per database:

- **Storage — $0.50 per provisioned GB, per month.** Billed on the size chosen, not the bytes written.
- **Active compute — $0.20 per vCPU, per hour.** Billed only while the database processes queries;
  an idle database carries no compute charge.

No charge for data transfer or for daily backups. **Dedicated pricing is quoted — contact support.**

## Choosing shared vs dedicated

- **Shared** — prototyping, or a production app with steady, moderate load. Storage is the customer's
  alone; compute is shared with other customers on the cluster, so performance is best-effort and there
  is **no uptime SLA**.
- **Dedicated** — isolated capacity, predictable performance under heavy or spiky load, a custom backup
  schedule and retention, or compliance that rules out shared infrastructure. Set up with the Forte
  team, not in the console.

## What to tell customers today

- Postgres is available to everyone now (open beta). The MongoDB-compatible engine is in closed alpha —
  request access from the engine tile on the new-database page.
- Create and manage databases with the **`forte databases` CLI** (`create --type postgres|mongodb`) or
  from `/console/projects/<projectId>/databases`. There is **no SDK** database API.
- Connect a database to services in the same project; the credentials and env vars are wired
  automatically and the service redeploys.
- Dedicated databases: contact support.

Docs: [Databases](https://forteplatforms.com/docs/databases) ·
[How databases work](https://forteplatforms.com/docs/databases/overview) ·
[Pricing](https://forteplatforms.com/docs/pricing)
