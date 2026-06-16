# CLI Reference

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

## Authentication Commands

Credentials are stored at `~/.forte/credentials.json` (owner-only permissions). They expire and are automatically re-prompted on next use.

| Command | Description |
|---|---|
| `forte login` (alias: `forte auth`) | Opens a browser window for OAuth authentication. |
| `forte logout` | Removes stored credentials. |
| `forte whoami` | Prints the authenticated account ID. Useful to verify auth is working. |

---

## Projects

```
forte projects list
forte projects get <projectId>
forte projects create <name>
forte projects update <projectId> [--google-oauth-client-id <clientId>]
forte projects delete <projectId> [--yes]
```

`--yes` skips the confirmation prompt on delete.

`--google-oauth-client-id` links a Google OAuth client to the project so end-users can log in. Required before accepting user logins.

---

## Services

Services are containerized apps deployed from a GitHub repository. Each gets a permanent HTTPS endpoint on `tryforte.dev`.

**Create**
```
forte services create <projectId> \
  --name <name> \
  --repo <githubHttpsUrl> \
  --branch <branch> \
  [--trigger push|release] \
  [--env KEY=VALUE ...] \
  [--auth-exclude <antPattern> ...]
```

| Flag | Default | Notes |
|---|---|---|
| `--trigger` | `push` | `push` deploys on every push to `--branch`; `release` deploys on GitHub releases. |
| `--branch` | required when `--trigger push` | Target branch for deployment. |
| `--env` | — | Set environment variables. Repeatable. `FORTE_` and `AWS_` prefixes are reserved. |
| `--auth-exclude` | — | Routes that bypass user auth. Ant patterns (`?`, `*`, `**`). Repeatable. |

Forte's GitHub App must be installed for the repo. If not: install at [github.com/apps/forte-platforms](https://github.com/apps/forte-platforms).

**List / Get / Delete**
```
forte services list <projectId>
forte services get <projectId> <serviceId>
forte services delete <projectId> <serviceId> [--yes]
```

**Update**
```
forte services update <projectId> <serviceId> \
  [--name <name>] \
  [--env KEY=VAL ...] \
  [--remove-env KEY ...] \
  [--auth-exclude <antPattern> ...] \
  [--instances <n>] \
  [--reset-dockerfile] \
  [--reset-healthcheck]
```

If the update triggers a new build, the CLI prints `"A new deployment has been triggered."` The Forte dashboard shows build status and logs.

`forte services list` shows `<owner>/<repo> · <branch> · <endpoint>` per service, so you can match a service to a local git repo.

---

## Requests

Inspect a service's per-request invocation logs (status, latency, exception). Services only — websites have no request logs. See `references/debugging.md` for the full "show me recent errors" flow.

```
forte requests <projectId> <serviceId> [options]            # list (default: recent 5xx)
forte requests get <projectId> <serviceId> <requestId>      # full detail for one request
```

| Flag | Default | Notes |
|---|---|---|
| `--status <codes...>` | common 5xx | Exact status code(s) to match, e.g. `--status 400 403`. The API has no status-class filter, so each code is a separate query. |
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
forte logs <projectId> <serviceId> [options]                       # list recent log lines
forte logs search <projectId> <serviceId> --search "<query>" [options]   # full-text search
```

| Flag | Default | Notes |
|---|---|---|
| `--request <requestId>` | — | Only log lines produced by this request (pairs with `forte requests`). |
| `--level <level>` | — | Filter by level: `ERROR`, `WARN`, `INFO`, ... |
| `--build <buildId>` | — | Only logs from this deployment/build. |
| `--search <query>` | — | Full-text query (required for the `search` action). |
| `--since <duration>` | `1h` | Look-back window: `30m`, `1h`, `24h`, `7d`. |
| `--limit <n>` | `50` | Max log lines to show. |
| `--json` | — | Output raw JSON instead of a table. |

---

## Proxy

Run a local reverse proxy that authenticates requests through Forte before forwarding to your local dev server.

```
forte proxy \
  [--project-id <id>] \
  [--service-id <id>] \
  [-p, --port <port>] \
  [--proxy-port <port>]
```

| Flag | Default | Notes |
|---|---|---|
| `--project-id` | interactive prompt | Project to authenticate against. |
| `--service-id` | interactive prompt | Service to authenticate against. |
| `-p, --port` | service's configured port, then `3000` | Port your local dev server is running on. |
| `--proxy-port` | `8080` | Port the proxy listens on. Auto-increments on `EADDRINUSE`. |

Proxy guide: [forteplatforms.com/docs/guides/testing-locally](https://forteplatforms.com/docs/guides/testing-locally)

---

## Websites

Websites are front-ends deployed from GitHub and served on a CDN. Use `forte websites` to manage them from the CLI.

**List / Get / Delete**
```
forte websites list <projectId>
forte websites get <projectId> <websiteId>
forte websites delete <projectId> <websiteId> [--yes]
```

**Create**
```
forte websites create <projectId> \
  --name <name> \
  --repo <githubHttpsUrl> \
  --branch <branch> \
  [--trigger push|release] \
  [--build-command <cmd>] \
  [--output-dir <dir>] \
  [--package-manager npm|yarn|pnpm|bun] \
  [--node-version <v>] \
  [--install-command <cmd>] \
  [--subdirectory <path>] \
  [--env KEY=VAL ...] \
  [--secret KEY=VAL ...]
```

Forte auto-detects the framework, build command, and output directory for most popular setups. `--branch` is required when `--trigger push` (the default). Secrets are encrypted at rest and never echoed back.

**Update** (PATCH — only supplied fields are changed)
```
forte websites update <projectId> <websiteId> \
  [--name <name>] \
  [--build-command <cmd>] \
  [--output-dir <dir>] \
  [--package-manager <pm>] \
  [--node-version <v>] \
  [--install-command <cmd>] \
  [--env KEY=VAL ...] \
  [--upsert-secret KEY=VAL ...] \
  [--remove-secret KEY ...] \
  [--reset-detected-config]
```

`--reset-detected-config` clears auto-detected build settings so Forte re-detects on the next push.

**Deploy** (trigger a new build / roll back to a specific commit)
```
forte websites deploy <projectId> <websiteId> [--commit <sha>]
```

When `--commit` is omitted, Forte redeploys the current branch HEAD.

Doc: [forteplatforms.com/docs/core-concepts/websites](https://forteplatforms.com/docs/core-concepts/websites)

---

## Common Errors

| Symptom | Cause | Fix |
|---|---|---|
| Error mentioning GitHub App | Forte can't access the repository | Install the GitHub App at [github.com/apps/forte-platforms](https://github.com/apps/forte-platforms) |
| Proxy fails to start | Port 8080 already in use | Use `--proxy-port 8081` (also auto-retries up to 5 times) |
| `forte whoami` fails or auth errors | Missing or expired credentials | Run `forte login` |
| `--branch` required error | `--trigger push` needs a branch | Add `--branch <branch>` |
