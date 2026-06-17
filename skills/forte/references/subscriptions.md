# Subscriptions

Forte's Subscriptions API charges end-users on a recurring **monthly or yearly** schedule through
Stripe, with Forte as the orchestration layer. You define an inline price (the same line items as a
one-off payment); Forte saves the card on the first charge, re-charges it off-session each cycle,
retries failures during a grace period, and keeps you in sync via triggers. **Every renewal is an
ordinary Forte `Payment`**, so existing payment reporting and `PAYMENT_COMPLETED` triggers just work.

## Prerequisites

- Same compliance/sandbox rules as Payments (live needs `APPROVED`; sandbox bypasses). See `payments.md`.
- **Early-access feature**: the account must have the `SUBSCRIPTIONS` feature enabled (request it from
  `forteplatforms.com/console/subscriptions`). Until granted, `createSubscription` returns
  `SUBSCRIPTIONS_ACCESS_REQUIRED`.

## Lifecycle

`state`: `INCOMPLETE` (first charge unconfirmed) → `ACTIVE` → `PAST_DUE` (a renewal failed; in the
grace window) → `CANCELLED` (terminal). The first charge is confirmed **on-session** with Stripe
Elements (and saves the card); renewals are **off-session** and automatic.

- `interval` is `MONTH` or `YEAR` only (charged once per interval — no "every N"). Other cadences:
  point the customer at support.
- `startTime` (optional, default now): a **future** start defers the first charge off-session to that
  time and **requires** a saved `paymentMethodId` (`SUBSCRIPTION_PAYMENT_METHOD_REQUIRED` otherwise).
- `endTime` (optional): the last charge is the last cadence point ≤ `endTime`, then it cancels. Omit =
  bill forever.

## Create + confirm

Client-side (`forte.users.createSubscription`) or server-side with `userId`
(`forte.projects.createSubscription`).

```typescript
const result = await forte.users.createSubscription({
  projectId,
  createSubscriptionRequest: {
    currency: "usd",
    interval: "MONTH",
    description: "Pro plan",
    lineItems: [{ description: "Pro plan", unitAmountCents: 4900, quantity: 1, taxCode: "txcd_10000000" }],
    customerAddress: { line1: "...", city: "...", state: "...", postalCode: "...", country: "US" },
    // paymentMethodId optional: omit to let the customer enter a card now (auto-saved for renewals).
  },
});
const { subscription, stripeClientSecret, stripePublishableKey, stripeConnectedAccountId } = result;
```

Confirm `stripeClientSecret` with Stripe Elements exactly as for a one-off payment (see `payments.md` —
`stripeAccount` is required). That single confirmation charges period 1 **and** saves the card. The
subscription flips `INCOMPLETE` → `ACTIVE` automatically. `createSubscriptionPreview` is the read-only
quote (returns totals + estimated `nextRenewalAt`); like create, it's on both surfaces
(`forte.users.createSubscriptionPreview` / `forte.projects.createSubscriptionPreview`).

## Manage

- **Change card**: `PATCH .../subscriptions/{id}` `{ paymentMethodId }` (re-validated as the user's own;
  if `PAST_DUE`, charges the new card immediately to recover).
- **Change price**: `PUT .../subscriptions/{id}/items` with the **full** `lineItems` list → staged,
  effective next renewal (current period frozen).
- **Cancel at period end**: `PATCH` `{ cancelAtPeriodEnd: true }`. **Cancel now**: `DELETE` (no refund of
  the current period). Acting on a `CANCELLED` sub → 400.
- **Dunning**: a failed renewal → `PAST_DUE` + retries over a grace window; updating the card recovers
  immediately; exhausting retries → `CANCELLED`. `paymentMethodExpiresBeforeNextRenewal` + the cached
  `cardBrand/cardLast4/cardExp*` let you warn before failure.
- **Refund**: refunding a subscription charge **cancels the subscription by default**
  (`keepSubscriptionActive=true` to keep billing); a dispute always cancels.

## Triggers (reuse Payment Triggers)

Every renewal is a `Payment`, so configure normal payment triggers. **One** new event,
`SUBSCRIPTION_STATUS_CHANGED`, covers the lifecycle moments payment events don't. **There is no
"subscription created" event** — activation is the first `PAYMENT_COMPLETED`.

| Event | When |
|---|---|
| `PAYMENT_COMPLETED` | First charge AND every renewal AND every recovery charge (grant/extend access). |
| `SUBSCRIPTION_STATUS_CHANGED` | Transition to `PAST_DUE` (prompt to fix card) or `CANCELLED` (revoke). `state` says which. |
| `PAYMENT_REFUNDED` | A subscription charge refunded/disputed via Stripe (same rules as payments). |

Exact sequences:
- **Create/activate** → one `PAYMENT_COMPLETED`. (No "created" event.)
- **Each renewal** → one `PAYMENT_COMPLETED`.
- **Renewal fails** → one `SUBSCRIPTION_STATUS_CHANGED` `state:"PAST_DUE"`, fired **once** on
  `ACTIVE→PAST_DUE` (not per retry). When a retry/card-fix succeeds → a single `PAYMENT_COMPLETED` (no
  separate "recovered" event).
- **Cancel** (any cause: DELETE, at-period-end, grace exhausted, `endTime`) → one
  `SUBSCRIPTION_STATUS_CHANGED` `state:"CANCELLED"`; plus `PAYMENT_REFUNDED` if a refund/dispute caused it.

Trigger body adds `event` (the event name), `subscriptionId` (on subscription events), and `state`
(`COMPLETED`/`REFUNDED`/`PAST_DUE`/`CANCELLED`); `paymentId` is on payment events. Renewal
`PAYMENT_COMPLETED` carries both `paymentId` and `subscriptionId`. Make handlers idempotent on
`(event, paymentId|subscriptionId, state)`. Delivery/headers/retries identical to payment triggers.

Canonical doc: [forteplatforms.com/docs/guides/creating-subscriptions](https://forteplatforms.com/docs/guides/creating-subscriptions)
