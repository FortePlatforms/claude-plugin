# CLI Reference

This is the **complete, authoritative** `forte` command surface. The CLI has **exactly 14 commands**: `login` (alias `auth`), `logout`, `whoami`, `projects`, `services`, `websites`, `requests`, `logs`, `payments`, `payment-methods`, `payment-triggers`, `actions`, `proxy`, `help`. There is **no** `forte init`, `forte deploy`, `forte build`, `forte test`, `forte web`, `forte run`, `forte env`, `forte secrets`, or `forte databases`. **Do not invent commands or flags** — if it is not listed here, it does not exist. Flag names are exact (e.g. it is `--output-dir`, not `--out-dir`; `--health-check-path`, not `--healthcheck`).

## Installation

**Homebrew (macOS & Linux — recommended)**
```bash
brew tap FortePlatforms/tap
brew install forte
# Upgrade: brew upgrade forte
```

**Shell script (Linux, macOS, Windows/Git Bash)**
```bash
curl -fsSL https://tryforte.dev/install.sh | bash
```
Installs to `~/.forte/bin` and adds it to PATH.

**Windows (native)** — download the binary from [github.com/FortePlatforms/homebrew-tap/releases](https://github.com/FortePlatforms/homebrew-tap/releases), rename to `forte.exe`, and add to PATH.

**Verify**: `forte --version`

Full install guide: [forteplatforms.com/docs/getting-started/installation](https://forteplatforms.com/docs/getting-started/installation)

---

## Global behavior (applies to every command)

- **Interactive pickers.** Most commands take resource IDs as **positional arguments**. When a required `projectId` / `serviceId` / `websiteId` / `userId` / `actionId` / etc. is **omitted**, the CLI drops into an interactive picker instead of erroring — so `forte services list` (no project) prompts you to pick a project. In non-interactive contexts (CI), pass the IDs explicitly. `forte projects create` similarly prompts for the name if you omit it.
- **First positional is usually the action.** Resource commands take the shape `forte <command> [action] [ids...]` where `action` defaults to `list`. For example `forte services` ≡ `forte services list`.
- **`--yes`** skips confirmation prompts on destructive actions (`delete`, `remove`, `cancel`, `refund`).
- **`--json`** (on `requests` and `logs`) prints raw JSON instead of a table.
- **Credentials** live at `~/.forte/credentials.json` (owner-only, mode `0600`). They expire and are automatically re-prompted via `forte login` on next use.
- **Environment selection.** The CLI targets Forte's production API by default. `FORTE_ENV` / `NODE_ENV` are development overrides used internally; customers don't need to set them.

---

## Authentication Commands

| Command | Args / Flags | Description |
|---|---|---|
| `forte login` (alias `forte auth`) | none | Opens a browser window for OAuth authentication. |
| `forte logout` | none | Removes stored credentials. |
| `forte whoami` | none | Prints the authenticated account ID. Useful to verify auth is working. |

### Terms of Service acceptance

Every Forte account must accept the Terms of Service before it can use the API. Accounts normally accept them on first sign-in to the console. An account created **during `forte login`** can reach the CLI without that step — so its first real command (for example `forte projects list`) fails with **HTTP 403, error code `TERMS_OF_SERVICE_NOT_ACCEPTED`**. This is not a permissions or "not logged in" problem. The CLI prints a clear message and opens the console so the account can review and accept the terms; once accepted, re-run the command. Re-running `forte login` also surfaces the terms step in the browser.

---

## Projects

```
forte projects [list]                                            # list all projects
forte projects get [projectId]                                   # show one project
forte projects create [name]                                     # create a project (prompts for name if omitted)
forte projects update [projectId] [--google-oauth-client-id <id>]
forte projects delete [projectId] [--yes]
```

| Flag | Applies to | Default | Notes |
|---|---|---|---|
| `--google-oauth-client-id <id>` | update | — | Links a Google OAuth client so end-users can log in. Required before accepting user logins. |
| `--yes` | delete | — | Skip the confirmation prompt. |

`forte projects list` shows every project and its ID; omit `projectId` on `get`/`update`/`delete` to pick interactively.

---

## Services

Services are containerized apps deployed from a GitHub repository. Each gets a permanent HTTPS endpoint on `tryforte.dev`. Forte **auto-detects** the Dockerfile (or generates one), the listening port, and the health-check path from your real code — see `references/build-and-detection.md`.

```
forte services [list] [projectId]
forte services get [projectId] [serviceId]
forte services create [projectId] --name <name> --repo <url> [--branch <branch>] [...]
forte services update [projectId] [serviceId] [...]
forte services delete [projectId] [serviceId] [--yes]
```

`forte services list` shows `<owner>/<repo> · <branch> · <endpoint>` per service, so you can match a service to a local git repo.

**Create** requires `--name` and `--repo`. Flags:

| Flag | Default | Notes |
|---|---|---|
| `--name <name>` | required | Service name. |
| `--repo <url>` | required | GitHub repository HTTPS URL. The Forte GitHub App must be installed for the repo. |
| `--branch <branch>` | required when `--trigger push` | Branch that triggers deploys. |
| `--trigger <push\|release>` | `push` | `push` deploys on every push to `--branch`; `release` deploys on GitHub releases. |
| `--base-dir <path>` | repo root | Base directory within the repo for monorepos. |
| `--cpu <vcpu>` | `0.25` | Container size in vCPU: `0.25`, `0.5`, `1`, or `2`. |
| `--instances <n>` | `1` | Base instances kept warm, 1–10. |
| `--health-check-port <n>` | auto-detected | Override the detected listening port. Usually unnecessary — Forte detects it. |
| `--health-check-path <path>` | auto-detected | Override the detected health-check path (must start with `/`). Usually unnecessary. |
| `--env KEY=VALUE ...` | — | Environment variables. Repeatable. `FORTE_` and `AWS_` prefixes are reserved. |
| `--secret KEY=VALUE ...` | — | Secrets, encrypted at rest, never echoed back. Repeatable. |
| `--enable-body-logging` | off | Capture request/response bodies in logs. |

**Update** — only supplied fields change. Flags:

| Flag | Notes |
|---|---|
| `--name <name>` | Rename the service. |
| `--branch <branch>` / `--trigger <push\|release>` | Change the deploy trigger. |
| `--base-dir <path>` | Change the monorepo base directory (empty string clears it). |
| `--cpu <vcpu>` / `--instances <n>` | Resize container / warm instances. |
| `--health-check-port <n>` / `--health-check-path <path>` | Override the detected port / path. |
| `--env KEY=VALUE ...` | **Replaces all** environment variables with the supplied set. (There is **no** `--remove-env`; to drop a var, re-supply the full set without it.) |
| `--upsert-secret KEY=VALUE ...` | Add or update secrets (does not touch other secrets). |
| `--remove-secret KEY ...` | Delete secrets by key. |
| `--auth-exclude <antPattern> ...` | Routes that bypass user auth. Ant patterns (`?`, `*`, `**`). Repeatable. |
| `--enable-body-logging` / `--disable-body-logging` | Toggle body logging. |
| `--reset-dockerfile` | Discard the cached Dockerfile so Forte re-detects/regenerates it on the next build. |
| `--reset-healthcheck` | Discard the cached port + health-check so Forte re-detects them on the next build. |

> Auth exclusions are applied via `forte services update --auth-exclude ...` — they are **not** read on `create`.

If the update triggers a new build, the CLI prints `"A new deployment has been triggered."` The Forte dashboard shows build status and logs.

---

## Websites

Websites are front-ends deployed from GitHub and served on a CDN. Forte **auto-detects** the framework, build command, output directory, package manager, and Node version from your repo (see `references/build-and-detection.md`); the overrides below are only needed when detection isn't right.

```
forte websites [list] [projectId]
forte websites get [projectId] [websiteId]
forte websites create [projectId] --name <name> --repo <url> [--branch <branch>] [...]
forte websites update [projectId] [websiteId] [...]
forte websites delete [projectId] [websiteId] [--yes]
forte websites deploy [projectId] [websiteId] [--commit <sha>]
forte websites domains <list|add|sync|remove> [projectId] [websiteId] [...]
forte websites deployments <list|get|logs|cancel> [projectId] [websiteId] [...]
```

**Create** requires `--name` and `--repo`. **Create / update** flags:

| Flag | Default | Notes |
|---|---|---|
| `--name <name>` | required (create) | Website name. |
| `--repo <url>` | required (create) | GitHub repository HTTPS URL. |
| `--branch <branch>` | required when `--trigger push` | Branch that triggers deploys. |
| `--trigger <push\|release>` | `push` | Build trigger. |
| `--build-command <cmd>` | auto-detected | Override the build command, e.g. `"npm run build"`. |
| `--output-dir <dir>` | auto-detected | Override the build output directory, e.g. `dist`. |
| `--package-manager <pm>` | auto-detected | `npm`, `yarn`, `pnpm`, or `bun`. |
| `--node-version <v>` | auto-detected | Node.js **major** version, e.g. `22`. |
| `--install-command <cmd>` | auto-detected | Override the dependency-install command. |
| `--subdirectory <path>` | repo root | Subdirectory within the repo to build from (monorepos). |
| `--env KEY=VALUE ...` | — | Environment variables. Repeatable. |
| `--secret KEY=VALUE ...` | — | Secrets, encrypted at rest. Repeatable (create/update). |

**Update-only** flags: `--upsert-secret KEY=VALUE ...`, `--remove-secret KEY ...`, and `--reset-detected-config` (clears the auto-detected build settings so Forte re-detects on the next push). Update is a PATCH — only supplied fields change.

**Deploy** — `--commit <sha>` deploys a specific commit (e.g. to roll back); when omitted, Forte redeploys the current branch HEAD.

**Custom domains** (`forte websites domains ...`):
```
forte websites domains list   [projectId] [websiteId]
forte websites domains add    [projectId] [websiteId] [domain] | [--domain <domain>]
forte websites domains sync   [projectId] [websiteId] [customDomainId]   # advance DNS/cert verification
forte websites domains remove [projectId] [websiteId] [customDomainId] [--yes]
```

**Deployment history** (`forte websites deployments ...`):
```
forte websites deployments list   [projectId] [websiteId]
forte websites deployments get    [projectId] [websiteId] [buildId]
forte websites deployments logs   [projectId] [websiteId] [buildId]
forte websites deployments cancel [projectId] [websiteId] [buildId] [--yes]
```

Doc: [forteplatforms.com/docs/core-concepts/websites](https://forteplatforms.com/docs/core-concepts/websites)

---

## Requests

Inspect a service's per-request invocation logs (status, latency, exception). Services only — websites have no request logs. See `references/debugging.md` for the full "show me recent errors" flow.

```
forte requests [list] [projectId] [serviceId] [options]      # default: recent 5xx
forte requests get [projectId] [serviceId] [requestId]       # full detail for one request
```

| Flag | Default | Notes |
|---|---|---|
| `--status <codes...>` | 5xx | Exact status code(s) to match, e.g. `--status 400 403`. Each code is a separate query (no status-class filter). |
| `--all` | — | List all requests regardless of status. |
| `--method <method>` | — | Filter by HTTP method. |
| `--path <path>` | — | Filter by exact request path. |
| `--since <duration>` | `24h` | Look-back window: `30m`, `1h`, `24h`, `7d`. |
| `--limit <n>` | `20` | Max requests to show. |
| `--json` | — | Output raw JSON instead of a table. |

`forte requests get` returns the exception type, message, and stack trace plus request/response bodies (when body logging is on). Each listed request shows its `requestId` — pass it to `forte requests get` or `forte logs --request`.

---

## Logs

Read a service's application log lines. Services only.

```
forte logs [list] [projectId] [serviceId] [options]                       # recent log lines
forte logs search [projectId] [serviceId] --search "<query>" [options]    # full-text search
```

| Flag | Default | Notes |
|---|---|---|
| `--request <requestId>` | — | Only log lines produced by this request (pairs with `forte requests`). |
| `--level <level>` | — | Filter by level: `ERROR`, `WARN`, `INFO`, ... |
| `--build <buildId>` | — | Only logs from this deployment/build. |
| `--search <query>` | — | Full-text query. **Required** for the `search` action. |
| `--since <duration>` | `1h` | Look-back window: `30m`, `1h`, `24h`, `7d`. |
| `--limit <n>` | `50` | Max log lines to show. |
| `--json` | — | Output raw JSON instead of a table. |

---

## Proxy

Run a local reverse proxy that authenticates requests through Forte before forwarding to your local dev server. Services only — websites are public-by-default and don't pass through Forte auth. See `references/auth-and-proxy.md`.

```
forte proxy [--project-id <id>] [--service-id <id>] [-p <port>] [--proxy-port <port>]
```

| Flag | Default | Notes |
|---|---|---|
| `--project-id <id>` | interactive prompt | Project to authenticate against. |
| `--service-id <id>` | interactive prompt | Service to authenticate against. |
| `-p, --port <n>` | service's detected port, else `3000` | Port **your local dev server** is running on. This is purely a local forwarding target — it has nothing to do with how Forte health-checks the deployed service. |
| `--proxy-port <n>` | `8080` | Port the proxy listens on. Auto-increments on `EADDRINUSE` (up to 5 times). |

Proxy guide: [forteplatforms.com/docs/guides/testing-locally](https://forteplatforms.com/docs/guides/testing-locally)

---

## Payments (beta)

Create and manage one-off payments for a project's end-users. Charges go through the user's saved payment method. See `references/payments.md`.

```
forte payments [list] [projectId] [userId] [--state <state>]
forte payments create  [projectId] [userId] [--item "<spec>" ...] [...] | [--json <body|@file>]
forte payments preview [projectId] [userId] [--item "<spec>" ...] [...] | [--json <body|@file>]
forte payments refund  [projectId] [userId] [--payment-id <id>] [--yes]
```

| Flag | Applies to | Notes |
|---|---|---|
| `--item "<description:unitAmountCents:quantity[:taxCode]>"` | create, preview | Line item. Repeatable. |
| `--currency <code>` | create, preview | 3-letter ISO code. Default `usd`. |
| `--description <text>` | create | Payment description. |
| `--metadata KEY=VALUE ...` | create | Arbitrary metadata. |
| `--customer-line1/-line2/-city/-state/-postal-code/-country <v>` | create, preview | Customer address (country is 2-letter). |
| `--shipping-line1/-line2/-city/-state/-postal-code/-country <v>` | create, preview | Shipping address (country is 2-letter). |
| `--json <value>` | create, preview | Full request body as JSON, or `@path/to/file.json`. Alternative to the per-field flags. |
| `--state <state>` | list | Filter by payment state. |
| `--payment-id <id>` | refund | Payment to refund (prompts among completed payments if omitted). |
| `--yes` | refund | Skip the confirmation prompt. |

`preview` calculates totals/tax without creating a charge.

---

## Payment methods (beta)

List and manage a user's saved payment methods. See `references/payments.md`.

```
forte payment-methods [list]        [projectId] [userId]
forte payment-methods set-default   [projectId] [userId] [--payment-method-id <id>]
forte payment-methods remove        [projectId] [userId] [--payment-method-id <id>] [--yes]
```

`--payment-method-id <id>` is optional — omit it to pick interactively. `--yes` skips the remove confirmation.

---

## Payment triggers (beta)

Register a service endpoint that Forte POSTs to when a payment event fires. See `references/payments.md`.

```
forte payment-triggers [list]   [projectId]
forte payment-triggers create   [projectId] --name <name> --service <serviceId> --path </path> [--events ... ] [--disabled]
forte payment-triggers update   [projectId] [triggerId] [...]
forte payment-triggers delete   [projectId] [triggerId] [--yes]
forte payment-triggers test     [triggerId]
```

| Flag | Notes |
|---|---|
| `--project-id <id>` | Supplies the project (skips the picker). Useful for `test`, whose first positional is the trigger ID. |
| `--name <name>` | Display name (create/update). |
| `--service <serviceId>` | Target service ID (create/update). |
| `--path <path>` | Target path on the service, e.g. `/webhooks/payment` (create/update). |
| `--events <event...>` | `PAYMENT_COMPLETED`, `PAYMENT_REFUNDED` (create/update). Repeatable. |
| `--enabled` / `--disabled` | Enable / disable the trigger. |
| `--yes` | Skip the delete confirmation. |

`forte payment-triggers test <triggerId>` sends a synthetic event to verify wiring.

---

## Actions (beta)

Schedule Forte to POST to one of your services on a cron schedule or once at a specific time. The target service must belong to the same project. See `references/actions.md`.

```
forte actions [list]       [projectId]
forte actions get          [projectId] [actionId]
forte actions create       [projectId] --name <name> --service <serviceId> --path </path> [schedule flags]
forte actions update       [projectId] [actionId] [...]
forte actions delete       [projectId] [actionId] [--yes]
forte actions invoke       [projectId] [actionId]                     # run once on demand
forte actions invocations  [projectId] [actionId]                     # list past invocations
forte actions cancel       [projectId] [actionId] [invocationId]      # cancel a pending invocation
```

Create requires `--name`, `--service`, and `--path`, plus schedule details.

| Flag | Notes |
|---|---|
| `--name <name>` | Action name (create/update). |
| `--service <serviceId>` | Target Forte service ID (create/update). |
| `--path <path>` | Path on the target service, e.g. `/jobs/nightly` (create/update). |
| `--body <body>` | Request body to POST (create/update). |
| `--schedule <recurring\|one-time>` | Schedule type. Default `recurring` (create). |
| `--cron <expr>` | Cron expression (recurring). Minimum cadence is once per minute. |
| `--timezone <tz>` | IANA timezone, e.g. `America/New_York` (recurring). |
| `--start <iso>` / `--end <iso>` | Recurrence window bounds, ISO 8601 (recurring). |
| `--at <iso>` | Run time for one-time actions, ISO 8601. |
| `--retryable` / `--disable-retry` | Enable retries (create/update) / disable retries (update). |
| `--enable` / `--disable` | Resume / pause the action (create can start paused). |
| `--yes` | Skip the delete confirmation. |

Examples:
```
forte actions create <projectId> --name nightly --service <serviceId> --path /jobs/nightly \
  --schedule recurring --cron "0 9 * * *" --timezone America/New_York
forte actions create <projectId> --name one-shot --service <serviceId> --path /jobs/run-once \
  --schedule one-time --at 2026-07-01T09:00:00Z
```

---

## Help

`forte help` (or `forte --help`, `forte <command> --help`) prints usage. Commander auto-generates per-command help listing the exact args, flags, and defaults — use it to confirm anything in this reference.

---

## Common Errors

| Symptom | Cause | Fix |
|---|---|---|
| Error mentioning the GitHub App | Forte can't access the repository | Install the GitHub App at [github.com/apps/forte-platforms](https://github.com/apps/forte-platforms) |
| Build fails with "no exposed port" / "no health check" / "framework not recognized" | Forte couldn't detect the port or a GET health-check route from the code | Expose a port (Dockerfile `EXPOSE` / listen on `$PORT`) and a GET health route, or set `--health-check-port` / `--health-check-path`. See `references/build-and-detection.md`. |
| Wrong build command or output dir on a website | Auto-detection picked the wrong framework setup | Override with `--build-command` / `--output-dir` (or `--reset-detected-config` to re-detect). |
| Proxy fails to start | Port 8080 already in use | Use `--proxy-port 8081` (also auto-retries up to 5 times) |
| `forte whoami` fails or auth errors | Missing or expired credentials | Run `forte login` |
| A `forte` command fails with `TERMS_OF_SERVICE_NOT_ACCEPTED` (HTTP 403) | The account hasn't accepted the Forte Terms of Service — common when the account was created during `forte login` | Accept the terms in the console (the CLI opens it for you), then re-run the command — or re-run `forte login`. |
| `--branch` required error | `--trigger push` needs a branch | Add `--branch <branch>` |
