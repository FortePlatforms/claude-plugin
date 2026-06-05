# API Surfaces — client-side vs server-side

Forte exposes **two** HTTP API surfaces. Which one you use depends on where the code runs and which
credential it holds. Getting this wrong is the most common — and most dangerous — Forte integration mistake.

|  | Client-side API | Server-side API |
|---|---|---|
| **SDK namespace** | `forte.users.*` | `forte.projects.*` |
| **Runs in** | Browser / mobile app | Your backend service |
| **Credential** | End-user session cookie (`Forte-User-Session-Token`) | `FORTE_API_TOKEN` |
| **Acts as** | The signed-in end user | The project owner |

## Client-side (`forte.users.*`)

Called directly from a browser or mobile app. Authenticates with the end-user's
`Forte-User-Session-Token` cookie, which Forte sets automatically on login. Scoped to the currently
signed-in user — one user can never read or modify another user's data.

```typescript
import { ForteClient } from "@forteplatforms/sdk";

// No credentials — relies on the Forte-User-Session-Token cookie
const forte = new ForteClient();

await forte.users.createPaymentPreview({ projectId, /* ... */ });
```

Use it to: sign users in (Google OAuth, OTP, password), read/update the signed-in user's profile,
manage and verify their own contact methods, create/list their own payments, manage their own content,
renew or invalidate their session.

From a **service** frontend, issue these calls same-site through the service's own origin —
`new ForteClient({ baseUrl: '/_forte' })` — so the `Forte-User-Session-Token` cookie stays
first-party. See `auth-and-proxy.md`.

## Server-side (`forte.projects.*`)

Called from your backend service. Authenticates with `FORTE_API_TOKEN`, which Forte injects
automatically as an environment variable into every deployed service. Acts as the project owner.

```typescript
import { ForteClient } from "@forteplatforms/sdk";

// FORTE_API_TOKEN is injected automatically inside a Forte service — no arguments needed
const forte = new ForteClient();

const { items } = await forte.projects.listUsers({ projectId });
```

Outside a Forte service (local dev, CI, external host), pass it explicitly:
```typescript
const forte = new ForteClient({ apiToken: process.env.FORTE_API_TOKEN });
```

Use it to: manage users as an admin (list, search, suspend, delete), read/set any user's contact
methods without a code, create payments on behalf of any user, manage service/website deployments,
read logs/metrics/build history, send email or SMS, create/revoke API keys.

## ⚠️ Credential safety — never ship `FORTE_API_TOKEN` to a browser

`FORTE_API_TOKEN` is a **server-side secret**. If it lands in a browser bundle or client-side JS,
any visitor can act as the project owner — read all user data, suspend accounts, send arbitrary
messages, and more. It belongs only in server-side code or backend environment variables, and is
intentionally **not** injected into Forte Websites for this reason.

The `Forte-User-Session-Token` cookie is the opposite: it's designed to live in the browser, scoped
to one user in one project, and carries no admin powers.

**Rule of thumb:** credential lives in a browser → `forte.users.*`. Credential lives in a server env
var → `forte.projects.*`. Need to act on another user's data → `forte.projects.*` (server-side only).

## Common pairings

Most resources expose both surfaces — a user-scoped path the end user calls themselves, and a
project-owner path your backend calls on their behalf.

| Resource | Client-side (`forte.users.*`) | Server-side (`forte.projects.*`) |
|---|---|---|
| Payments | `createPayment` | `createPayment` (takes `userId`) |
| Payment previews | `createPaymentPreview` | `createPaymentPreview` (takes `userId`) |
| Contact methods | self-service add / verify / delete | admin add, override, mark verified without code |
| Refunds | — (no client-side route) | `refundPayment` |
| File content | own content only | any user's content |

Canonical doc: [forteplatforms.com/docs/core-concepts/api-surfaces](https://forteplatforms.com/docs/core-concepts/api-surfaces)

---

## Sandbox (test) mode

A project-level flag set **at creation time** in the Forte Console — it is **permanent and immutable**
(a sandbox project can't become live, or vice versa). Use a separate project per environment.

A sandbox project is a full-fidelity test double: users authenticate, services respond, payments
process — but nothing has real-world consequences.

| Capability | Live project | Sandbox project |
|---|---|---|
| Hard-delete users | ❌ (suspend only) | ✅ immediate & permanent |
| Override contact methods / mark verified without a code | ❌ | ✅ |
| Request/response body logging default | off | on |
| Payments | real money | Stripe **test mode** (use Stripe test cards; publishable key is `pk_test_...`) |

Sandbox projects also **bypass the compliance gate** for payments, so you can build against test mode
before finishing onboarding. Do **not** use sandbox for production traffic, real billing, real
compliance data, or real customer PII.

Canonical doc: [forteplatforms.com/docs/core-concepts/sandbox-mode](https://forteplatforms.com/docs/core-concepts/sandbox-mode)
