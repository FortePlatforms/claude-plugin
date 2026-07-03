# Debugging — "show me my recent Forte errors"

When the user asks something like *"show me my recent Forte errors for the service this
project deploys to"* or *"why is my deployed API failing?"*, run this loop. The whole thing
is `forte` CLI calls plus a little reasoning — no dashboard required.

Only **services** have request and application logs. Websites bypass the Forte gateway, so
they have no request logs or metrics — if the repo is a website, say so and stop.

## 1. Identify the repo

```bash
git remote get-url origin        # -> https://github.com/<owner>/<repo>
git rev-parse --abbrev-ref HEAD  # current branch (helps disambiguate environments)
```

Reduce the remote to `<owner>/<repo>`.

## 2. Find the matching service

Environments are **separate projects** (e.g. `<app>-staging`, `<app>-production`), so a repo
can map to more than one service.

```bash
forte projects list                 # all projects (id + name)
forte services list <projectId>     # each row shows: <owner>/<repo> · <branch> · <endpoint>
```

Match `<owner>/<repo>` against the service rows **in memory** across the candidate projects.
- One match → use it.
- Several matches (e.g. staging + production) → use **AskUserQuestion** to let the user pick
  the environment. Default the question toward production, since that's usually where the user
  cares about errors.

If you've resolved these before, you can cache the project/service IDs to Claude memory (the
skill already suggests this for project IDs) so future sessions skip steps 1–2. Verify a cached
ID still resolves before relying on it.

## 3. List the failing requests

`forte requests` defaults to recent **server errors (5xx)** over the last 24h — the common case:

```bash
forte requests <projectId> <serviceId> --json
```

The API filters by an **exact** status code, so `forte requests` queries the common 5xx codes
for you. For other classes, name the codes explicitly (each is a separate query):

```bash
forte requests <projectId> <serviceId> --status 400 403 --since 24h --json   # client errors
forte requests <projectId> <serviceId> --all --since 1h --json               # everything, recent
```

Useful flags: `--since 30m|1h|24h|7d` (default `24h`), `--method`, `--path`, `--limit`
(default 20), `--json`. Use `--json` so you can read `requestId`, `statusCode`, latency, and
`exceptionType` reliably.

**Clients get a 503 with error code `SERVICE_PAUSED` but `forte requests` shows nothing:** the
service is paused — Forte rejects the request before it reaches the service, so no request log is
written. `forte services get <projectId> <serviceId>` shows a `Paused` row (the console service
page shows a "Paused" badge). Resume the service from the console service page, or trigger a new
deployment — deploys auto-resume a paused service — then re-test.

## 4. Drill into one failing request

Each request log carries the request id. Get the full record (exception type/message/stack
trace and request/response bodies):

```bash
forte requests get <projectId> <serviceId> <requestId> --json
```

And read the application log lines that same request produced:

```bash
forte logs <projectId> <serviceId> --request <requestId> --level ERROR --json
```

For a broader hunt, full-text search the logs:

```bash
forte logs search <projectId> <serviceId> --search "<text>" --level ERROR --since 1h --json
```

## 5. Explain and fix

Summarize what's failing (which route, what status, the exception), point at the offending
code in the repo when the stack trace names a file, and propose a concrete fix. If the user
wants, apply it and let the next push redeploy.

See `references/cli.md` for the full flag reference for `forte requests` and `forte logs`.
