# API

The server exposes a REST API under `/api`, following the layered request
flow in [architecture.md](./architecture.md): `routes` → `middleware` →
`controllers` → `services` → `models`.

## Conventions

- **Versioned base path**: every route lives under `/api` (e.g.
  `/api/products`, `/api/cart`, `/api/orders`). If a breaking change is ever
  needed, it gets a new prefix (`/api/v2`) rather than changing `/api` in
  place.
- **Resources are plural nouns**: `/api/products`, `/api/orders`,
  `/api/cart` — not verbs in the path (`/api/getProducts`) and not singular
  (`/api/product`).
- **Standard verbs map to actions** on a resource:

  | Verb | Path | Action |
  | --- | --- | --- |
  | `GET` | `/api/products` | List products |
  | `GET` | `/api/products/:id` | Get one product |
  | `POST` | `/api/products` | Create a product (admin) |
  | `PATCH` | `/api/products/:id` | Update a product (admin) |
  | `DELETE` | `/api/products/:id` | Delete a product (admin) |
  | `GET` | `/api/cart` | Get the current user's cart |
  | `POST` | `/api/cart/items` | Add an item to the cart |
  | `PATCH` | `/api/cart/items/:itemId` | Update a cart line item (e.g. quantity) |
  | `DELETE` | `/api/cart/items/:itemId` | Remove a cart line item |
  | `POST` | `/api/orders` | Place an order (checkout) |
  | `GET` | `/api/orders` | List the current user's orders (admin: all orders) |
  | `GET` | `/api/orders/:id` | Get one order |
  | `PATCH` | `/api/orders/:id` | Update order status (admin) |

  A verb is never overloaded to mean two different actions on the same path
  (e.g. `POST /api/orders` is always "create," never also "update" via a body
  flag).

- **Status codes are used correctly**, not defaulted to `200`/`500` for
  everything:

  | Code | Meaning | When |
  | --- | --- | --- |
  | `200` | OK | Successful `GET`/`PATCH`/`DELETE` |
  | `201` | Created | Successful `POST` that creates a resource (signup, create product, place order) |
  | `400` | Bad Request | Request fails validation (bad body/params) |
  | `401` | Unauthorized | Missing or invalid token (see [auth.md](./auth.md)) |
  | `403` | Forbidden | Valid token, but the role/ownership doesn't permit the action |
  | `404` | Not Found | Resource doesn't exist (or, for ownership-scoped resources, doesn't exist *for this user*) |
  | `409` | Conflict | Uniqueness violation (e.g. duplicate email on signup, duplicate slug) |
  | `500` | Internal Server Error | Unexpected/unhandled failure |

## Response shape

Every response — success or failure — follows one shape:

```ts
// Success
{
  "success": true,
  "data": /* resource or array of resources */
}

// Failure
{
  "success": false,
  "error": {
    "message": string
  }
}
```

There is no response that omits `success`, mixes `data` and `error`, or
returns a bare array/object at the top level. This shape is produced in one
place — a shared response helper used by controllers (e.g. `sendSuccess(res,
data, status?)` / `sendError(res, message, status)`) — and a centralized
error-handling middleware catches thrown/unhandled errors and formats them
into the same `error` shape, so controllers never construct the failure
shape ad hoc or leak a raw stack trace to the client.

## Controllers and services

- **Controllers stay thin**: parse/validate the request (params, query,
  body — validated against the Zod schemas in `/shared` where the shape is
  shared with the client), call a service function to do the actual work,
  and shape the service's result into the standard response. A controller
  does not contain business logic (pricing, stock checks, order totals,
  etc.) or talk to a model directly.
- **Services hold the logic**, are reusable across controllers, and are
  where the actual model queries happen. A service function raises/returns
  a typed error (e.g. `NotFoundError`, `ValidationError`) rather than
  formatting an HTTP response itself — shaping the response is the
  controller's job.

## Auth requirements per route group

Enforced via the `authenticate` / `isAdmin` middleware from
[auth.md](./auth.md), composed as `authenticate` → `isAdmin` →
controller for admin-only routes:

| Route group | Example paths | Requirement |
| --- | --- | --- |
| Public | `GET /api/products`, `GET /api/products/:id` | None — no token required |
| Customer | `GET/POST/PATCH/DELETE /api/cart*`, `POST /api/orders`, `GET /api/orders` (own), `GET /api/orders/:id` (own) | Valid token (`authenticate`); queries scoped to `req.user.id` |
| Admin | `POST/PATCH/DELETE /api/products`, `GET /api/orders` (all), `PATCH /api/orders/:id` (status) | Valid token *and* `role === 'admin'` (`authenticate` + `isAdmin`) |

Note `GET /api/orders` appears in both the customer and admin rows: for a
customer-authenticated request it returns only their own orders (scoped by
`user`); for an admin it returns all orders (unscoped) — same path, behavior
branches on `req.user.role`, per the access pattern in
[database.md](./database.md#access-patterns). This is the one deliberate
exception to "one verb+path, one action" above, and it stays valid only
because the branch is on the authenticated role, not on client input.
