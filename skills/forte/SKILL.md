---
name: forte
description: Use when the user mentions Forte, the forte CLI, forteplatforms.com, deploying with Forte, forte proxy, forte sessions, FORTE_API_TOKEN, forte projects, forte services, or @forteplatforms/sdk. Also use when setting up a new app for deployment and the repo contains forte-sdk or references tryforte.dev, or when asked to show recent Forte errors, debug a failing request, or pull request/application logs for a deployed service.
---

# Forte

Forte is a deployment and end-user authentication platform. This skill keeps Claude accurate on the real CLI surface, prevents invented commands, and routes to canonical references and docs.

## Hard Rules

These invariants are commonly violated — do not cross them:

- **The CLI has exactly 14 commands**: `login` (alias: `auth`), `logout`, `whoami`, `projects`, `services`, `websites`, `requests`, `logs`, `payments`, `payment-methods`, `payment-triggers`, `actions`, `proxy`, `help`. There is **no** `forte init`, `forte deploy`, `forte build`, `forte test`, `forte web`, `forte run`, `forte env`, `forte secrets`, or `forte databases`. Do not invent commands or flags — `references/cli.md` is the exact, complete surface (positional args, flag names, defaults). Flag names are precise: it's `--output-dir` not `--out-dir`, `--health-check-path` not `--healthcheck`, and there is **no** `--remove-env` (service env is replaced wholesale via `--env`).
- **The CLI writes nothing to the user's repo.** No `forte.json`, no `.forterc`. IDs are CLI arguments — surface them with `forte projects list`, `forte services list <projectId>`, or `forte websites list <projectId>`.
- **Forte projects contain three deployable resources**: services (containerized backends), websites (front-ends from GitHub, served on a CDN), and content. Services and websites are different primitives — see `references/setup-walkthrough.md` for which to recommend.
- **Forte detects deploy config from your real code — it does not assume language defaults.** At build time Forte inspects the actual repository to determine the listening port, the health-check path, the build/start commands, the package manager, the runtime version, and whether to use the repo's Dockerfile or generate one. **Never tell a customer "Forte defaults to port 3000 for Node" (or any per-language port) for deployment.** There is no such deploy-time default: the port and health-check path come from the real Dockerfile `EXPOSE` and the app's real HTTP routes, and if Forte can't determine them with confidence the **build fails with a specific, actionable error** rather than silently guessing. (The `3000`/`8000`/`8080` numbers exist only as a last-resort local forwarding target for `forte proxy` — unrelated to deployment.) Detected values are overridable and re-detectable (services: `--health-check-port`/`--health-check-path`/`--reset-healthcheck`/`--reset-dockerfile`; websites: `--build-command`/`--output-dir`/`--package-manager`/`--node-version`/`--reset-detected-config`). See `references/build-and-detection.md`.
- **Managed Databases (early access) are a project-level resource.** Forte offers managed **PostgreSQL** and **MongoDB** that belong to a *project* (not to a service or a signed-in user) and connect automatically and securely to the services you attach them to — Forte injects the connection string into the service as an environment variable, so nothing is wired up by hand. They are encrypted by default and HIPAA-ready, with automated backups and the ability to branch ephemeral test databases from real data. **There is no `forte databases` CLI command and no SDK database API yet** — during early access, databases are created and managed from the console only; do not tell customers to run `forte databases ...`. Customers request access from the Databases tab in the console (`/console/databases`). See `references/databases.md`.
- **Three end-user auth methods are live**: Google OAuth, OTP (one-time passcode over email or SMS), and password sign-in — plus optional **multi-factor authentication** (a second factor: authenticator app/TOTP, passkeys/WebAuthn, email or SMS OTP, or backup codes), configured per project. Password and per-channel OTP login are toggled on in project settings; when MFA is enabled, login responses carry an `mfaStatus` and may return a short-lived pending token (usable only on the MFA endpoints until the second factor is verified). See `references/auth-and-proxy.md`.
- **Sandbox (test) mode exists** as a project-level flag set **at creation time** — it is permanent and immutable (a project can't switch between live and sandbox). It unlocks hard-delete, contact-method overrides, default body logging, and Stripe test-mode payments. For *local* testing of a live service, use `forte proxy` (services only — websites are public-by-default and don't go through Forte auth). Multiple environments = multiple projects. See `references/api-surfaces.md`.
- **`forte.users.*` (browser, session cookie) and `forte.projects.*` (backend, `FORTE_API_TOKEN`) are two distinct API surfaces.** Never put `FORTE_API_TOKEN` in browser/client code. See `references/api-surfaces.md`.
- **Request logs and application logs are available from the CLI** (and the Forte dashboard). `forte requests` lists per-request invocation logs (status, latency, exception) and `forte requests get` shows one request in full; `forte logs` reads application log lines, filterable by request id, level, or full-text search. Both take `--json` for machine-readable output. To investigate "show me recent errors", follow `references/debugging.md`. Only **services** produce these — websites bypass the gateway, so they have no request logs or metrics.
- **Forte Actions (beta) call one of your Forte services on a schedule.** `forte actions ...` registers a target **service + path** (like a payment trigger) that Forte sends a POST to — either on a recurring cron schedule (with a timezone) or once at a specific time. The target service must belong to the same project. Each execution is recorded as an *invocation*; actions can be marked retryable, paused/resumed, invoked on demand, and have pending invocations cancelled. Minimum cadence is once per minute. See `references/actions.md`.
- **Service env vars `FORTE_PROJECT_ID`, `FORTE_SERVICE_ID`, `FORTE_API_TOKEN` are injected automatically** inside Forte-hosted services. Websites do NOT receive these — websites are public front-ends and don't authenticate against Forte. Do not use the `FORTE_` or `AWS_` prefix for custom env vars — both are reserved.
- **Website traffic never passes through the Forte gateway — this includes server-rendered (SSR) websites.** Services are gateway-proxied: Forte authenticates every request (injecting the `X-Forte-User-Id` header), enforces auth path exclusions, and captures per-request logs and metrics. Websites — static *and* SSR — are served directly from the CDN, so **none** of that applies: no Forte auth enforcement, no `X-Forte-User-Id`, no request logs, no metrics. An SSR website's server code runs, but Forte never sees or authenticates the individual request, so the site must enforce its own auth. For authenticated, observable request handling, deploy a **service** and call it from the website. See `references/setup-walkthrough.md`.
- **Same-site session cookies on services use the reserved `/_forte` path.** For browser-direct `forte.users.*` calls from a service's own frontend, point the SDK at the service's origin via `new ForteClient({ baseUrl: '/_forte' })` so the `Forte-User-Session-Token` cookie stays first-party. Services only — websites bypass the gateway, so `/_forte` doesn't exist there. See `references/auth-and-proxy.md`.

## Quick Command Reference

```
forte login                                                           # authenticate (opens browser)
forte whoami                                                          # verify auth
forte projects list                                                   # list all projects
forte projects create <name>                                          # create a project
forte services list <projectId>                                       # list services (shows repo · branch)
forte services create <projectId> --name <n> --repo <url> --branch <b>
forte requests <projectId> <serviceId>                               # recent failing requests (5xx by default)
forte requests <projectId> <serviceId> --status 400 403 --since 24h  # specific status codes
forte requests get <projectId> <serviceId> <requestId>              # full request detail (exception, bodies)
forte logs <projectId> <serviceId> --request <requestId> --level ERROR   # logs one request produced
forte logs search <projectId> <serviceId> --search "<text>"        # full-text log search
forte websites list <projectId>                                       # list websites
forte websites create <projectId> --name <n> --repo <url> --branch <b>
forte websites deploy <projectId> <websiteId> [--commit <sha>]       # trigger/rollback deploy
forte actions list <projectId>                                       # list actions (beta)
forte actions create <projectId> --name <n> --service <serviceId> --path </p> --schedule recurring --cron "0 9 * * *" --timezone <tz>
forte actions create <projectId> --name <n> --service <serviceId> --path </p> --schedule one-time --at <iso8601>
forte actions invoke <projectId> <actionId>                          # run once on demand
forte actions invocations <projectId> <actionId>                     # list past invocations
forte payments list <projectId> <userId>                             # list a user's payments (beta)
forte payment-methods list <projectId> <userId>                      # list a user's saved payment methods — cards & bank accounts (beta)
forte payment-triggers list <projectId>                              # list payment-event webhooks (beta)
forte proxy [--project-id <id>] [--service-id <id>] [-p <port>]     # local dev proxy (services only)
```

Omitting a required `projectId` / `serviceId` / `websiteId` / `userId` drops into an interactive picker. `references/cli.md` is the complete, exact command + flag surface — consult it before using any flag.

## When to Read Which Reference

| Situation | Reference |
|---|---|
| Installing the CLI, login errors, credential issues, a `TERMS_OF_SERVICE_NOT_ACCEPTED` 403 | `references/cli.md` |
| "Show me recent errors" / "debug a failing request" / pulling request or application logs for a service | `references/debugging.md` |
| "Set up Forte for my app" / "deploy this" / creating projects, services, or websites | `references/setup-walkthrough.md` |
| How Forte builds/detects an app — port, health-check, Dockerfile, build command, framework; build-detection failures; "what port will it use?" | `references/build-and-detection.md` |
| Managed databases — Postgres/Mongo, attaching to services, ephemeral test data (early access) | `references/databases.md` |
| "How does auth work", "how do sessions work", "proxy not working", OTP / password login | `references/auth-and-proxy.md` |
| Client-side vs server-side API, where `FORTE_API_TOKEN` is safe, sandbox/test mode | `references/api-surfaces.md` |
| Charging users, payment previews, refunds, Stripe Elements, payment methods (card & ACH direct debit), off-session charges, payment triggers | `references/payments.md` |
| Recurring billing — monthly/yearly subscriptions, renewals, dunning/grace, cancellation, subscription triggers | `references/subscriptions.md` |
| Scheduling URL calls on a cron/timer or one-time, retries, invocations | `references/actions.md` |
| Adding the SDK, making API calls from app code, managing websites via SDK | `references/sdk-integration.md` |
| HIPAA, BAA, SOC 2, compliance, certifications, BAA requests | `references/compliance.md` |
| Pointing the customer to official docs for a topic | `references/docs-index.md` |

## Defer-Don't-Invent

When a question goes beyond what the references cover, link the customer to the relevant `forteplatforms.com/docs/*` page rather than guessing. The docs are the canonical source of truth.
