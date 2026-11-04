# Checkout vs. one-time purchase vs. subscription

Forte has three ways to start charging an end-user. Pick by what the **first charge** must contain.

| The customer is buying… | Use | Creates |
|---|---|---|
| One-time items only (a one-time purchase) | `createPayment` | One `Payment` |
| One plan, with nothing else on the first charge | `createSubscription` | One `Subscription`, plus a `Payment` per charge |
| One-time items **and** a plan, or several plans, in one charge | `createCheckout` | One `Payment` and N `Subscription`s |

| | `createPayment` | `createSubscription` | `createCheckout` |
|---|---|---|---|
| Recurring | No | Yes, one schedule | Each subscription on its own schedule; one-time lines never renew |
| Card | Yes | Yes | Yes |
| ACH direct debit | Opt-in | No | No (`CHECKOUT_REQUIRES_CARD` for a saved bank account) |
| Future start (`startTime`) | No | Yes | Yes, per subscription: charged separately on that date |
| Off-session first charge | Yes (`offSession`) | Yes (future `startTime`) | Only for future-dated subscriptions |
| Limits | | | 10 subscriptions, 50 lines in total |

Common mistakes:

- **Don't call `createPayment` and then `createSubscription` for a setup fee plus a plan.** That is two charges and two confirmations. Use `createCheckout`.
- **Don't put a one-time fee in `createSubscription`'s `lineItems`.** Every line on a subscription renews, so the fee is charged again each cycle. Use `createCheckout`.
- **Don't look for a checkout object.** `createCheckout` stores no checkout: there is no checkout ID and no get, list, cancel, or refund route. Afterward, manage the `Payment` and each `Subscription` with their normal APIs (`references/payments.md`, `references/subscriptions.md`).

The console shows a mixed checkout as a normal payment and labels each line **One-time purchase** or **Subscription**. The CLI has no checkout command, and neither do the MCP tools. Checkout is API and SDK only.

## Prerequisites

Same compliance and sandbox rules as Payments (`references/payments.md`).

## Create + confirm

Client-side, `forte.users.createCheckout` (`POST /api/v1/{projectId}/users/me/checkouts`). Server-side with `userId`, `forte.projects.createCheckout` (`POST /api/v1/projects/{projectId}/users/{userId}/checkouts`).

```typescript
const result = await forte.users.createCheckout({
  projectId,
  createCheckoutRequest: {
    currency: "usd",
    description: "Pro plan with onboarding",
    lineItems: [
      { description: "Onboarding", unitAmountCents: 9900, quantity: 1, taxCode: "txcd_20030000" },
    ],
    subscriptions: [
      {
        interval: "MONTH",
        description: "Pro plan",
        lineItems: [{ description: "Pro plan", unitAmountCents: 4900, quantity: 1, taxCode: "txcd_10000000" }],
      },
    ],
    customerAddress: { line1: "...", city: "...", state: "...", postalCode: "...", country: "US" },
    // paymentMethodId optional: omit to let the customer enter a card now (saved for renewals).
  },
});
const { payment, subscriptions, stripeClientSecret, stripePublishableKey, stripeConnectedAccountId } = result;
```

- Each subscription takes `interval` (`MONTH` or `YEAR`), `lineItems`, and optional `description`, `metadata`, `startTime` (must be in the future, else `CHECKOUT_START_TIME_INVALID`), and `endTime` (must be in the future and after `startTime`, else `CHECKOUT_END_TIME_INVALID`).
- `subscriptions` in the response matches request order, and each starts `INCOMPLETE`. It is empty for a one-time-only checkout.
- `metadata` on the checkout is stored on the `Payment`.
- Forte sets `lineItemId` and `subscriptionId` on lines and ignores any you send.
- Send at least one one-time line or one subscription (`CHECKOUT_EMPTY`), and no more than 50 lines (`CHECKOUT_TOO_MANY_LINE_ITEMS`).
- The subtotal due today, when anything is due today, and each subscription's subtotal must each clear the minimum charge, because each subscription renews as its own charge.

Confirm `stripeClientSecret` once with Stripe Elements, exactly as for a one-off payment (see `payments.md`; `stripeAccount` is required). That one confirmation pays every line, saves the card, and flips every subscription to `ACTIVE`. This applies to server-side calls and pinned saved cards too: the customer still confirms in the browser. If a saved card can't be reused for renewals, each subscription is cancelled and the one-time items stay paid.

An unconfirmed checkout is abandoned: its `DRAFT` payment is cancelled after 24 hours, and its `INCOMPLETE` subscriptions are cancelled too. Create a new checkout rather than reusing the client secret.

## Future start

A subscription with `startTime` isn't on today's charge. The customer enters a card once but sees two charges: today's, then a separate off-session charge on `startTime`. Tell them before they pay.

- It stays `INCOMPLETE` until that first charge succeeds, and a decline follows the usual `PAST_DUE` path. Grant access on its own `PAYMENT_COMPLETED`: today's `PAYMENT_COMPLETED` doesn't list it in `subscriptionIds`.
- If today's charge is never confirmed, it's cancelled and never charged.
- With nothing due today (every subscription future-dated, no one-time lines), `payment` is `null`. With `paymentMethodId`, `stripeClientSecret` is `null` and nothing needs confirming. Without it, confirm `stripeClientSecret` with `stripe.confirmSetup`, not `confirmPayment`.
- Cancel before the start with `DELETE`, or `PATCH { cancelAtPeriodEnd: true }` to cancel on `startTime` without charging.
- Free trial: set `startTime` to the trial's end, with nothing else on the checkout. There's no trial setting and no trigger when it starts, so grant trial access yourself when `createCheckout` returns (saved card) or `confirmSetup` succeeds (new card), not on `PAYMENT_COMPLETED`, which first fires when the trial ends.

## Preview

`createCheckoutPreview` takes the same body minus `paymentMethodId` and is read-only (`POST .../checkouts/preview` on both surfaces). It returns the first charge (`subtotalCents`, `taxCents`, `amountCents`, `lineItems`, excluding future-dated subscriptions) and a `subscriptions` list with each renewal's `interval`, totals, `lineItems`, `startTime`, and `nextRenewalAt`. Use it for "Due today: $X. Then $49.00 per month." Renewal figures are estimates, because each renewal recalculates tax. `taxCents` is `null` with no address.

## Renewals

Each subscription renews on its own interval and charges only its own lines. A $99 setup fee plus a $49 monthly plan renews at $49 plus tax. After the first charge, each subscription behaves like any other (`references/subscriptions.md`).

## Tax addresses

- Subscription lines are always taxed at `customerAddress`, on the first charge and on renewals.
- One-time lines are taxed at `shippingAddress` if sent, otherwise `customerAddress`.
- `shippingAddress` requires `customerAddress` (`CHECKOUT_SHIPPING_REQUIRES_CUSTOMER_ADDRESS`).
- With no `customerAddress`, no tax is computed.

## Triggers

A checkout fires **one** `PAYMENT_COMPLETED`, not one per item. The body includes `subscriptionIds`, with every subscription the payment started, and also `subscriptionId` when there is exactly one. A checkout with no subscriptions omits `subscriptionIds`. Each later renewal fires its own `PAYMENT_COMPLETED` with that subscription's `subscriptionId`.

Read `subscriptionIds` in new handlers. The body carries no line items, so record what a payment grants in your backend, keyed by `payment.id`, when you create the checkout.

## Refunds

Use `refundPayment` on the checkout's `Payment` (server-side only). Pass `createPaymentRefundRequest.lineItems` to refund individual lines by `lineItemId`. Per line, `amountCents` is the **pre-tax** amount; omit it to refund the rest of the line. Forte computes the tax share. Don't combine `lineItems` with `refundAmountCents` (`400`).

| Refund | Default effect on subscriptions | With `keepSubscriptionActive=true` |
|---|---|---|
| Lines through the API or console | Cancels only those whose lines were refunded | Cancels none |
| Amount only through the API or console | Cancels all on the payment | Cancels none |
| Partial refund in the Stripe dashboard | Cancels none | Not applicable |
| Full refund in the Stripe dashboard, or a lost dispute | Cancels all | Not applicable |

Refunding a one-time line leaves subscriptions running. Refunding lines never cancels a future-dated subscription, since it has no lines on today's charge; the rows that cancel all include it.

Canonical doc: [forteplatforms.com/docs/guides/checkout](https://forteplatforms.com/docs/guides/checkout)
