# Databases (early access)

Forte offers **managed databases** so customers don't operate a database themselves. A database belongs
to a **project** (like services and websites), not to a single service or a signed-in user. Forte
provisions it, generates the credentials, encrypts it, and backs it up daily.

> **Early access.** Managed databases are gated. Customers enable them by contacting support. Once
> enabled, databases are managed either from the **Databases** tab of a project in the console —
> `/console/projects/<projectId>/databases` — or from the **`forte databases` CLI command** (see
> `references/cli.md` for the exact surface). There is still **no SDK database API**; do not call or
> invent an SDK database method.

## What is actually available today

- **PostgreSQL only.** MongoDB is **coming soon** — the API rejects it today. Customers can register
  interest from the Databases tab. Never tell a customer Mongo is available now.
- **Shared tier only, self-serve.** The **dedicated** tier is **contact support** — it is not something
  a customer can create in the console.

## How it fits together

- **Project-scoped.** A database lives at the project level. Every service in the same project can
  connect to it. Separate environments are separate projects, so each environment gets its own database.
- **Automatic connection wiring.** Connecting a database to a service creates a Postgres role dedicated
  to that service, sets the environment variable names the customer chooses (a single `DATABASE_URL` by
  default, or discrete host/port/database/username/password), and redeploys the service. Forte generates
  the password and **never shows it** — not in the console, not in the API, not in the service's env var
  list. Disconnecting drops the role, removes the variables, and redeploys.
- **Encrypted.** Storage is encrypted; connections require TLS (`sslmode=require`).
- **Storage-only sizing.** 10–250 GB, presets at 10 (Hobby), 25 (Standard), 100 GB (Performance).
  Resize is online and instant, and can shrink as well as grow — but not below the data already stored.
  There is **no compute sizing on the shared tier**; compute is shared across the cluster.
- **Daily backups.** Restore is a support request. Do **not** promise point-in-time recovery, read
  replicas, high availability, or a recovery-point objective.
- **Storage enforcement.** Forte emails at 80% and 95% full. At 100% the database goes **read-only** —
  reads work, writes are rejected. Resizing fixes it immediately; deleting data requires a temporary
  write window from support, because cleanup itself needs writes.

## Limits worth knowing

| Limit | Value |
|---|---|
| Databases per account | 5 (1 without a verified billing method) |
| Storage per database | 10–250 GB |
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

- Postgres is available now; MongoDB is coming soon.
- Enable early access, then create databases from `/console/projects/<projectId>/databases`.
- Connect a database to services in the same project; the credentials and env vars are wired
  automatically and the service redeploys.
- There is no CLI or SDK for databases yet — use the console.
- Dedicated databases: contact support.

Docs: [Databases](https://forteplatforms.com/docs/databases) ·
[How databases work](https://forteplatforms.com/docs/databases/overview) ·
[Pricing](https://forteplatforms.com/docs/pricing)
