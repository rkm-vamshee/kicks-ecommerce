# Payments

Kicks supports two payment methods: **Razorpay** (online payment) and
**cash on delivery (COD)**. An order's payment is never marked `paid` on the
client's say-so alone — for Razorpay that confirmation comes from a verified
webhook, and for COD it's a deliberate admin action after delivery.

## Order status vs. payment status

Order status and payment status are **separate fields** on the `Order`
model (see [database.md](./database.md#order)) — a single "status" field is
not overloaded to mean both. An order's `status` describes fulfillment
progress; a separate `paymentStatus` field describes whether it's been paid.

Order `status` flows through a fixed set of states:

```
pending → paid → processing → shipped → delivered
                                              \
                                        cancelled (from any pre-delivered state)
```

**Note:** [database.md](./database.md#order) currently defines `Order.status`
as `["pending", "paid", "shipped", "delivered", "cancelled"]`, without
`processing`. That enum needs `processing` added to match the flow
described here — treat this doc as the source of truth for the state list
and update the schema to match, rather than silently dropping `processing`.

`paymentStatus` is its own small enum, independent of `status` — e.g.
`["pending", "paid"]` — and moves according to the payment method described
below, on its own schedule relative to `status`.

## Razorpay

1. **Server creates a payment intent.** When checkout is initiated, the
   server calls the Razorpay API to create an order/payment intent
   server-side (never client-side, since it requires the secret key) and
   returns the intent id / client-facing details to the client. The
   corresponding Kicks `Order` is created with `status: "pending"` and
   `paymentStatus: "pending"`.
2. **Client confirms the payment** using Razorpay's client-side checkout
   integration (referred to here as "Razorpay Elements" per the product
   requirement — Razorpay's actual SDK is the Razorpay Checkout modal, not a
   component library, but the flow is the same: the client never handles
   raw card details itself, Razorpay's hosted UI does). This step only
   captures the customer's payment method and hands it to Razorpay — it does
   not, by itself, mark anything paid in Kicks.
3. **The order is marked paid only after Razorpay confirms via webhook** —
   never from the client's confirmation callback alone. The client-side
   "success" callback is treated as informational (e.g. "show a
   processing/confirming state to the user") and never as the trigger that
   sets `paymentStatus: "paid"` or `status: "paid"`. A client could always
   claim success without a real payment having gone through, so only the
   server-to-server webhook is trusted.
4. **Webhook handler**: a dedicated route (e.g. `POST
   /api/webhooks/razorpay`) receives Razorpay's webhook event, verifies it
   (below), looks up the corresponding `Order` by the payment/order id in
   the payload, and — for a successful-payment event — sets `paymentStatus:
   "paid"` and advances `status` from `pending` to `paid`. A failed-payment
   event leaves the order as-is (or marks it in a way the customer can retry
   checkout), but never marks it paid.

### Webhook signature verification

Razorpay webhook signatures are **always verified**, with no exception:

- Every request to the webhook route is verified using Razorpay's HMAC
  signature scheme — the raw request body is hashed with the webhook secret
  and compared to the `X-Razorpay-Signature` header — before the payload is
  trusted or acted on.
- The webhook route needs the **raw** request body for this comparison, so
  it's excluded from (or ordered before) any body-parsing middleware that
  would transform it before the signature check runs.
- A request with a missing or invalid signature is rejected (`400`) and
  never reaches the logic that updates an order — an attacker who knows an
  order id cannot mark it paid by posting a forged payload to the webhook
  route.
- The webhook secret is a server-only environment variable, never exposed
  to the client (distinct from any client-facing Razorpay key used to
  initialize checkout in step 2).

## Cash on delivery (COD)

- When COD is selected at checkout, the order is **placed immediately**:
  `status: "pending"`, `paymentStatus: "pending"`. There is no intent
  creation or webhook involved — placing the order is the entire payment
  step from the customer's side.
- The order is **marked paid by an admin, after delivery** — an explicit
  admin action (e.g. `PATCH /api/orders/:id` setting `paymentStatus:
  "paid"`, per [api.md](./api.md)) once cash has actually been collected.
  This is a manual, human-triggered transition, not something the system
  infers from `status` reaching `delivered` — a delivered COD order is not
  automatically paid until an admin confirms it.

## Summary

| Method | Who marks `paymentStatus: "paid"` | Trigger |
| --- | --- | --- |
| Razorpay | Server (webhook handler) | Verified webhook event from Razorpay, never the client |
| COD | Admin | Explicit admin action after delivery is confirmed |

In both cases, the party that can mark an order paid is a trusted
server-side actor (a verified webhook or an authenticated admin) — never the
customer-facing client on its own.
