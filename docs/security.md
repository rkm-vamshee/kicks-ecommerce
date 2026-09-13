# Security

This doc collects the security practices that apply across the app. Several
are covered in depth elsewhere; this is the checklist that ties them
together.

## Secrets

- Secrets (`JWT_SECRET`, `MONGO_URI`, Razorpay keys/webhook secret, etc.)
  live only in environment variables — never hardcoded in source, never
  committed to the repo.
- Each package's real `.env` is gitignored (see `.gitignore`:
  `.env`, `.env.local`, `.env.*.local`), while `.env.example` is checked in
  as a template listing the variable names a deployment needs, with
  placeholder/non-secret values — e.g. `server/.env.example` and
  `client/.env.example`. Adding a new required env var means updating the
  relevant `.env.example`, not just a local `.env`.
- No secret is ever logged, returned in an API response, or exposed to the
  client — this includes `JWT_SECRET` and the Razorpay webhook secret, which
  are server-only and never sent to or read by client code (distinct from
  any client-facing Razorpay publishable key used to initialize checkout,
  per [payments.md](./payments.md#razorpay)).

## CORS

- The server restricts CORS to the **known client origin** — configured via
  `CLIENT_URL` (`server/.env.example`) and passed as the `origin` option to
  the `cors` middleware — not a wildcard (`*`) or a reflected/permissive
  origin. Only the app's own client is allowed to make cross-origin requests
  to the API.

## Passwords

- Passwords are hashed with bcrypt before storage and compared with
  `bcrypt.compare()` at login; plaintext passwords are never stored, logged,
  or returned. Full detail in [auth.md](./auth.md#password-storage).

## Rate limiting

- Auth routes (`/api/auth/login`, `/api/auth/signup`) and checkout
  (`/api/orders` creation, and by extension the Razorpay intent-creation
  step) are rate-limited to stop brute-force login attempts and abusive
  order/payment-intent creation. Applied as middleware on those specific
  routes (e.g. via `express-rate-limit`), not uniformly across the whole
  API — routes that don't touch credentials or payment creation don't need
  the same limit. Exact limits (requests per window): **TBD**.

## Security headers

- `helmet` is applied to the Express app (near the top of the middleware
  stack, alongside `cors` and the JSON body parser in
  `server/src/index.ts`) to set standard security headers (e.g.
  `X-Content-Type-Options`, `X-Frame-Options`,
  `Strict-Transport-Security`) rather than leaving them unset or
  hand-rolling them per response.

## JWT secrets

- `JWT_SECRET` is a strong, random value (not a short or guessable string),
  set via environment variable per [Secrets](#secrets) above.
- If a `JWT_SECRET` is ever suspected compromised, it is rotated: a new
  secret is set and the server redeployed. Rotating the secret invalidates
  every previously issued token (since none will verify against the new
  secret), which is the intended effect — a compromised secret is treated as
  fully burned, not patched around.

## Razorpay webhooks

- Every Razorpay webhook request's signature is verified before its payload
  is trusted or used to update an order — no exception, and never
  short-circuited in any environment. Full detail, including the
  raw-body requirement, in
  [payments.md](./payments.md#webhook-signature-verification).

## User-generated content

- Any user-generated content rendered in the client (product reviews,
  display names, or anything else a user can type) is escaped on render,
  not inserted as raw HTML. React's default JSX rendering (`{value}`)
  already escapes text content, so this holds automatically as long as
  user content is rendered through normal JSX interpolation — `
  dangerouslySetInnerHTML` (or an equivalent raw-HTML insertion) is not
  used on user-generated content.
