# Docs Index

Use this to find the right link to give a customer when their question isn't fully covered by the other references. All URLs are under `https://forteplatforms.com/docs/`.

| Topic | URL path | What it covers |
|---|---|---|
| CLI installation | `getting-started/installation` | Homebrew, curl, binary downloads, `forte login` |
| 5-minute deploy quickstart | `getting-started/quickstart` | Create project → service → first deploy |
| What is a project? | `core-concepts/projects` | Isolation, unlimited free projects, staging vs production pattern |
| Client-side vs server-side API | `core-concepts/api-surfaces` | `forte.users.*` vs `forte.projects.*`, credential safety, choosing the right surface |
| What is a service? | `core-concepts/services` | GitHub-backed containerized deploys, permanent HTTPS endpoint, auth path exclusions, injected env vars |
| What is a website? | `core-concepts/websites` | GitHub-backed front-end deploys, build configuration, framework auto-detection, public URL on `sites.tryforte.dev` |
| What is an action? | `core-concepts/actions` | Scheduled/recurring URL calls, one-time vs cron, retries, invocations, trusted-request headers (beta) |
| Sandbox (test) mode | `core-concepts/sandbox-mode` | Create-time-only immutable test projects, hard-delete, contact overrides, Stripe test mode |
| SDKs (install + first call) | `core-concepts/sdks` | TS/Python/Java install, client creation, common use cases, custom attributes |
| User model | `core-concepts/users` | User states, attributes, lifecycle overview |
| End-user authentication | `users/authentication` | Google OAuth, OTP (email/SMS) login, reCAPTCHA |
| Password sign-in & reset | `users/passwords` | Password login, strength rules, change, reset modes, error codes |
| Contact method verification | `users/contact-methods` | OTP flows, 6-digit codes, resend rules, expiry, reclaim |
| Session tokens | `users/sessions` | Token format, 365-day default, renewal, logout |
| User administration | `users/administration` | Active/Suspended/Deleted states, audit trail |
| Deploying services | `guides/deploying-services` | Full build pipeline, Dockerfile auto-detection, build timeout, GitHub status |
| Deploying websites | `guides/deploying-websites` | Step-by-step website deploy flow, framework auto-detection, build configuration overrides |
| Creating payments | `guides/creating-payments` | Preview vs create, choosing methods (card/ACH), Stripe Elements, off-session charges, refunds, payment triggers, compliance |
| ACH direct debit | `guides/ach-direct-debit` | Accept bank-account payments, instant vs microdeposit verification, async settlement & states, saving a bank account, off-session charges |
| Creating actions | `guides/creating-actions` | Registering a scheduled/one-time URL call, testing with invoke-now, viewing invocations (beta) |
| Local proxy testing | `guides/testing-locally` | `forte proxy` setup, LAN testing on mobile (services only) |
| Monitoring | `guides/monitoring` | Log viewing and metrics from the Forte dashboard |
| Monorepo setup | `guides/monorepo` | Deploying one service out of a monorepo |
| Pricing | `pricing` | Free tier, limits |

When in doubt, link to [forteplatforms.com/docs](https://forteplatforms.com/docs) and let the customer navigate.
