# Errors and Validation

## Validation

- All request input — body, params, query — is validated with Zod **at the
  route boundary**, before the request ever reaches a controller. Validation
  is a `validate(schema)` middleware inserted between the route path and the
  controller (`router.post('/products', authenticate, isAdmin,
  validate(createProductSchema), createProduct)`), not a check performed
  inside the controller itself. A controller can assume its input already
  matches the schema by the time it runs.
- Schemas live in `/shared` and are imported by both `client` and `server`,
  per [coding-standards.md](./coding-standards.md#shared-types) — the same
  `createProductSchema` that gates the server route also validates the
  admin form on the client before it ever submits. This is what keeps
  client-side and server-side validation from drifting apart into two
  different sets of rules.
- A schema validates the shape the client/server boundary actually needs
  (a request body, e.g. `CreateProductInput`), not the full persisted model —
  server-only fields (`_id`, `createdAt`, computed values) are never part of
  the input schema a client is expected to satisfy.

## Validation failures → 400

- When `validate(schema)` fails, it responds `400` immediately (the request
  never reaches the controller) with the standard failure shape from
  [api.md](./api.md#response-shape), where `error` carries enough structure
  for the client to render the failure **inline, next to the relevant
  field** — not just a single flat message:

  ```ts
  {
    "success": false,
    "error": {
      "message": "Validation failed",
      "fields": {
        "email": "Invalid email address",
        "price": "Must be greater than 0"
      }
    }
  }
  ```

  `fields` keys match the schema's field names one-to-one, so the client can
  look up `error.fields[fieldName]` while rendering a form without any
  string-matching against a prose message.
- This is produced by formatting Zod's own issue list (`error.issues`) into
  `{ field: message }` in the `validate` middleware — it is not hand-written
  per route, so every route's validation failures come back in the same
  shape automatically.

## Unexpected errors

- A single Express error-handling middleware, registered last (after every
  route), is the only place that turns a thrown/unhandled error into an HTTP
  response. Controllers and services throw real `Error` objects (per
  [coding-standards.md](./coding-standards.md#errors)) and let them propagate
  — they don't catch-and-respond individually — so this middleware is the one
  place that decides what the client sees for anything unexpected.
- On the server, it logs full context: the error's message and stack trace,
  the request method/path, and (when available) the authenticated
  `req.user.id` — enough to debug the failure from logs alone.
- To the client, it returns the standard failure shape with a **safe,
  generic message** (`"Something went wrong"`) and `500`, unless the error is
  a known, typed error (e.g. `NotFoundError` → `404` with a message like
  `"Product not found"`, per [api.md](./api.md#conventions)) that is safe to
  surface as-is. The raw `error.message` and `error.stack` from an
  *unexpected* error are never sent to the client — only errors explicitly
  designed to be user-facing (typed domain errors with a deliberately
  written message) reach the response body.
- This applies regardless of environment. Development and production return
  the same safe shape to the client; the difference is only in what's
  logged/visible on the server (e.g. more verbose console output in dev),
  never in what the response body contains.

## Summary of the flow

```
request
  → validate(schema)   -- 400 + { fields } on bad input, controller never runs
  → controller          -- assumes input is valid
  → service              -- throws typed errors (NotFoundError, etc.)
  → error middleware     -- logs full context, responds with safe message + correct status
```
