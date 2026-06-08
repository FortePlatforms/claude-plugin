---
name: forte
description: Use when the user mentions Forte, the forte CLI, forteplatforms.com, deploying with Forte, forte proxy, forte sessions, FORTE_API_TOKEN, forte projects, forte services, or @forteplatforms/sdk. Also use when setting up a new app for deployment and the repo contains forte-sdk or references tryforte.dev.
---

# Forte

Forte is a deployment and end-user authentication platform. This skill keeps Claude accurate on the real CLI surface, prevents invented commands, and routes to canonical references and docs.

## Hard Rules

These invariants are commonly violated — do not cross them:

- **The CLI has exactly 9 commands**: `login` (alias: `auth`), `logout`, `whoami`, `projects`, `services`, `websites`, `actions`, `proxy`, `help`. There is **no** `forte init`, `forte deploy`, `forte build`, `forte logs`, `forte test`, `forte web`, `forte run`. Do not invent them.
- **The CLI writes nothing to the user's repo.** No `forte.json`, no `.forterc`. IDs are CLI arguments — surface them with `forte projects list`, `forte services list <projectId>`, or `forte websites list <projectId>`.
- **Forte projects contain three deployable resources**: services (containerized backends), websites (front-ends from GitHub, served on a CDN), and content. Services and websites are different primitives — see `references/setup-walkthrough.md` for which to recommend.
- **Managed Databases (early access) are a project-level resource.** Forte offers managed **PostgreSQL** and **MongoDB** that belong to a *project* (not to a service or a signed-in user) and connect automatically and securely to the services you attach them to — Forte injects the connection string into the service as an environment variable, so nothing is wired up by hand. They are encrypted by default and HIPAA-ready, with automated backups and the ability to branch ephemeral test databases from real data. **There is no `forte databases` CLI command and no SDK database API yet** — during early access, databases are created and managed from the console only; do not tell customers to run `forte databases ...`. Customers request access from the Databases tab in the console (`/console/databases`). See `references/databases.md`.
- **Three end-user auth methods are live**: Google OAuth, OTP (one-time passcode over email or SMS), and password sign-in. Password and per-channel OTP login are toggled on in project settings. See `references/auth-and-proxy.md`.
- **Sandbox (test) mode exists** as a project-level flag set **at creation time** — it is permanent and immutable (a project can't switch between live and sandbox). It unlocks hard-delete, contact-method overrides, default body logging, and Stripe test-mode payments. For *local* testing of a live service, use `forte proxy` (services only — websites are public-by-default and don't go through Forte auth). Multiple environments = multiple projects. See `references/api-surfaces.md`.
- **`forte.users.*` (browser, session cookie) and `forte.projects.*` (backend, `FORTE_API_TOKEN`) are two distinct API surfaces.** Never put `FORTE_API_TOKEN` in browser/client code. See `references/api-surfaces.md`.
- **Log viewing is done from the Forte dashboard.** There is no CLI command for logs.
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
forte services list <projectId>                                       # list services
forte services create <projectId> --name <n> --repo <url> --branch <b>
forte websites list <projectId>                                       # list websites
forte websites create <projectId> --name <n> --repo <url> --branch <b>
forte websites deploy <projectId> <websiteId> [--commit <sha>]       # trigger/rollback deploy
forte actions list <projectId>                                       # list actions (beta)
forte actions create <projectId> --name <n> --service <serviceId> --path </p> --schedule recurring --cron "0 9 * * *" --timezone <tz>
forte actions create <projectId> --name <n> --service <serviceId> --path </p> --schedule one-time --at <iso8601>
forte actions invoke <projectId> <actionId>                          # run once on demand
forte actions invocations <projectId> <actionId>                     # list past invocations
forte proxy [--project-id <id>] [--service-id <id>] [-p <port>]     # local dev proxy (services only)
```

## When to Read Which Reference

| Situation | Reference |
|---|---|
| Installing the CLI, login errors, credential issues | `references/cli.md` |
| "Set up Forte for my app" / "deploy this" / creating projects, services, or websites | `references/setup-walkthrough.md` |
| Managed databases — Postgres/Mongo, attaching to services, ephemeral test data (early access) | `references/databases.md` |
| "How does auth work", "how do sessions work", "proxy not working", OTP / password login | `references/auth-and-proxy.md` |
| Client-side vs server-side API, where `FORTE_API_TOKEN` is safe, sandbox/test mode | `references/api-surfaces.md` |
| Charging users, payment previews, refunds, Stripe Elements, payment triggers | `references/payments.md` |
| Scheduling URL calls on a cron/timer or one-time, retries, invocations | `references/actions.md` |
| Adding the SDK, making API calls from app code, managing websites via SDK | `references/sdk-integration.md` |
| Pointing the customer to official docs for a topic | `references/docs-index.md` |

## Defer-Don't-Invent

When a question goes beyond what the references cover, link the customer to the relevant `forteplatforms.com/docs/*` page rather than guessing. The docs are the canonical source of truth.
