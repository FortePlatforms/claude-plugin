# Payments

Forte's Payments API charges your end-users through Stripe, with Forte as the orchestration layer
(it handles webhooks, tax calculation, and chargebacks). Your company becomes the merchant of record
after completing **Compliance Registration** in the console.

## Prerequisites

- The project owner has completed compliance registration (`forteplatforms.com/console/compliance`).
- **Live** projects require compliance in the `APPROVED` state, or Forte rejects payment calls with
  `PAYMENTS_REQUIRE_APPROVED_COMPLIANCE`. **Sandbox** projects bypass this gate and run against
  Stripe Test mode — no separate flag, routing is automatic. See `api-surfaces.md` for sandbox mode.

## Two operations × two surfaces

| Operation | Purpose | Side effects |
|---|---|---|
| `createPaymentPreview` | Compute subtotal, tax, total **without** charging. Call repeatedly as the user edits the cart. | None — nothing persisted. |
| `createPayment` | Create a Forte `Payment` + Stripe PaymentIntent, ready to confirm with Stripe Elements. | Persists a payment, creates the PaymentIntent + tax calculation, writes the user's audit trail. |

Each has a client-side route (signed-in user pays for themselves) and a server-side route (your
backend charges a user it authenticated some other way). See `api-surfaces.md` for the full model.

- Client-side: `forte.users.createPaymentPreview` / `forte.users.createPayment`
- Server-side: `forte.projects.createPaymentPreview` / `forte.projects.createPayment` (both take `userId`)

## Preview (read-only, for cart pages)

```typescript
import { ForteClient } from "@forteplatforms/sdk";

const forte = new ForteClient(); // browser: relies on the session cookie

const preview = await forte.users.createPaymentPreview({
  projectId,
  createPaymentPreviewRequest: {
    currency: "usd",
    lineItems: [
      { description: "Pro plan", unitAmountCents: 2500, quantity: 1, taxCode: "txcd_10000000" },
    ],
    customerAddress: {
      line1: "354 Oyster Point Blvd", city: "South San Francisco",
      state: "CA", postalCode: "94080", country: "US",
    },
  },
});

const { subtotalCents, taxCents, amountCents } = preview;
```

- Response: `subtotalCents`, `taxCents`, `amountCents`, `currency`, and per-line `taxAmountCents`.
- Omit `customerAddress` → no tax computed, `amountCents == subtotalCents`.
- `taxCode` is a Stripe product tax category (e.g. `txcd_99999999` general goods, `txcd_10000000`
  digital goods). Tax is always **exclusive** (added on top of the unit price).

## Create (the "Pay" click)

Same payload, optionally with `description`. The `userId` in the URL (server-side) must match the
user being charged.

```typescript
const result = await forte.users.createPayment({
  projectId,
  createPaymentRequest: {
    currency: "usd",
    description: "May 2026 invoice",
    lineItems: [
      { description: "Pro plan", unitAmountCents: 2500, quantity: 1, taxCode: "txcd_10000000" },
    ],
    customerAddress: { line1: "...", city: "...", state: "...", postalCode: "...", country: "US" },
  },
});

const { payment, stripeClientSecret, stripePublishableKey, stripeConnectedAccountId } = result;
```

The `payment` object includes `id`, `state` (starts `DRAFT`), `amountCents`, `stripePaymentIntentId`.

## Confirm in the browser with Stripe Elements

Pass the three Stripe identifiers from the response straight to your frontend. **`stripeAccount` is
required** on `loadStripe` — these are direct charges on the connected account, not the platform.

```typescript
import { loadStripe } from "@stripe/stripe-js";

const stripe = await loadStripe(stripePublishableKey, { stripeAccount: stripeConnectedAccountId });
const elements = stripe.elements({ clientSecret: stripeClientSecret });
elements.create("payment").mount("#payment-element");

await stripe.confirmPayment({ elements, confirmParams: { return_url: "..." } });
```

Without `stripeAccount`, confirmation fails with "no such PaymentIntent."

## Payment lifecycle

Forte maps Stripe PaymentIntent status to a `Payment.state` and updates it automatically via webhooks:
`DRAFT` → `PROCESSING` → `COMPLETED` (or `CANCELLED` / `FAILED`). Refunds move it to `REFUNDED`.

A `Payment` left unconfirmed in `DRAFT` for 24 hours is automatically cancelled (state → `CANCELLED`,
and its PaymentIntent is cancelled too, so it can't be confirmed afterward — create a new payment
instead of reusing the old client secret).

## Refunds — backend only

`refundPayment` issues a **full** refund of a `COMPLETED` payment and sets state to `REFUNDED`.
There is **no client-side route** — only:

```
POST /api/v1/projects/{projectId}/users/{userId}/payments/{paymentId}/refund
```

```typescript
const forte = new ForteClient({ apiToken: process.env.FORTE_API_TOKEN });

// 1. Revoke the customer's entitlements in YOUR backend FIRST.
await revokeEntitlements(userId, paymentId);
// 2. Then refund through Forte.
const payment = await forte.projects.refundPayment({ projectId, userId, paymentId });
```

⚠️ A refund you initiate does **not** fire a `PAYMENT_REFUNDED` trigger — nothing calls back into
your system, so you must reverse whatever the payment granted before refunding. Only `COMPLETED`
payments are refundable (`PAYMENT_NOT_REFUNDABLE` otherwise; `PAYMENT_ALREADY_REFUNDED` if already done).

## Payment triggers (webhooks to your services)

When a payment changes state, Forte POSTs to one of your Forte Services over a private network. Use
it for fulfillment side effects (licenses, receipts, ledger writes).

- Events: `PAYMENT_COMPLETED`, and `PAYMENT_REFUNDED` — the latter **only** for refunds/chargebacks
  Forte detects via Stripe (e.g. a Stripe-dashboard refund or a lost dispute), **not** for refunds you
  initiate via `refundPayment`.
- Configured per project in the console (Project settings).
- Each delivery sends headers `Content-Type: application/json` and `X-Forte-Trusted: 1`, with body
  `{ userId, paymentId, paymentTime, state }`.
- **Validate the `X-Forte-Trusted` header** — Forte strips it from any other request, so its presence
  proves the call is genuine.
- Retry policy: 5 attempts, 60s apart, 30s timeout; success = HTTP 2xx. Make handlers **idempotent on
  `(paymentId, state)`** and return 2xx in under 30s.

Test a handler against your local dev server (real payload, real `X-Forte-Trusted` header):
```bash
forte payment-triggers test            # interactive picker
forte payment-triggers test pmt_trigger_<id>
```

Canonical doc: [forteplatforms.com/docs/guides/creating-payments](https://forteplatforms.com/docs/guides/creating-payments)
