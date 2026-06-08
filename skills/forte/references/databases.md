# Databases (early access)

Forte offers **managed databases** — **PostgreSQL** and **MongoDB** — so customers don't operate a
database themselves. A database belongs to a **project** (like services and websites), not to a single
service or a signed-in user. Forte provisions, encrypts, backs up, and scales it.

> **Early access / closed alpha.** Managed Databases are gated. Customers request access from the
> **Databases** tab in the console (`/console/databases`). There is **no `forte databases` CLI command
> and no SDK database API yet** — during early access, databases are created and managed from the
> console only. Do not tell customers to run `forte databases ...` or call an SDK database method;
> point them to the console.

## How it fits together

- **Project-scoped.** A database lives at the project level. Every service in the same project can be
  attached to it. Separate environments are separate projects, so each environment gets its own
  isolated database.
- **Automatic, secure connection.** When a database is attached to a service, Forte connects them over
  the private network and injects the connection string into the service as an environment variable at
  runtime. The customer does not copy a connection string around or manage credentials by hand.
- **Encrypted and HIPAA-ready.** Data is encrypted at rest and in transit by default; suitable for
  regulated workloads, with per-project data isolation.
- **Managed operations.** Forte handles automated backups, compute autoscaling, and storage scaling.
- **Ephemeral test databases.** Forte can branch ephemeral databases from real data, giving isolated,
  true-to-production test environments that spin up and tear down on demand.

## What to tell customers today

- Postgres and MongoDB are both available in early access.
- Request access from the console: `/console/databases`.
- Attach a database to the services in the same project; the connection is wired automatically.
- For provisioning there is no CLI or SDK yet — use the console.

Canonical product page: [forteplatforms.com/databases](https://forteplatforms.com/databases)
