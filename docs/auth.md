# Auth

Kicks uses JWT-based authentication. The server issues a signed token on
login and signup; the client stores it and sends it on every request; server
middleware verifies it and attaches the authenticated user to the request.
Two roles exist — `customer` and `admin` — and authorization for each is
enforced server-side, never trusted from the client.

## Password storage

- Passwords are hashed with bcrypt before being persisted — the `User` model
  never stores a plaintext password (see the `password` field in
  [database.md](./database.md)).
- Hashing happens once, at signup (and on password change), via a
  pre-save hook or an explicit `bcrypt.hash()` call in the auth service —
  never in a controller, so there's no path that writes a `User` document
  without hashing first.
- Login compares the submitted password against the stored hash with
  `bcrypt.compare()`; the plaintext password is never logged, returned in a
  response, or stored anywhere, even temporarily.

## Token issuance

- On successful signup or login, the server signs a JWT with the user's
  `id` and `role` as payload (`jwt.sign({ id, role }, JWT_SECRET, { expiresIn:
  JWT_EXPIRES_IN })`), using the `JWT_SECRET` and `JWT_EXPIRES_IN` from
  `server/.env.example`.
- The token is returned to the client in the response body (not set as a
  server-managed cookie), alongside the non-sensitive user profile (id,
  name, email, role) — never the password hash.
- The payload carries only what authorization checks need (`id`, `role`).
  It is not a place to stash other user data, since anything in it is
  readable by the client and stale until the token is reissued.

## Client storage and usage

- The client stores the token (mechanism — `localStorage` vs. an in-memory
  store — is an implementation detail of the client's auth module) and
  attaches it to every authenticated request as an `Authorization: Bearer
  <token>` header.
- A shared API client (not each call site individually) is responsible for
  attaching the header, so there's a single place that knows how the token
  is stored and sent.
- On a `401` response the client treats the token as invalid/expired, clears
  it, and redirects to login rather than retrying the request with the same
  token.

## Server-side verification

- An `authenticate` middleware (in `server/middleware`) reads the
  `Authorization` header, verifies the JWT with `JWT_SECRET`, and attaches
  the decoded `{ id, role }` to `req.user`. If the header is missing or the
  token fails verification (bad signature, expired), the middleware responds
  `401` and the request never reaches the controller.
- Route handlers and services read the authenticated user from `req.user`
  only — never from a request body or query param — so a client can't
  impersonate another user by passing a different id.

## Role-based access

Two roles: `customer` (default) and `admin`, stored on the `User` model
(see [database.md](./database.md#user)).

- **Customer routes** require only a valid token — `authenticate` runs, and
  any authenticated user (customer or admin) may proceed. These routes also
  scope their queries by `req.user.id` for ownership (e.g. a customer's own
  cart/orders), per the access pattern in [database.md](./database.md#access-patterns).
- **Admin routes** require a valid token *and* the admin role. An `isAdmin`
  middleware runs after `authenticate` and checks `req.user.role ===
  'admin'`, responding `403` if not. This check happens in server middleware
  only — the server never infers or accepts a role from anything the client
  sends (headers, body, query params); the only source of truth for role is
  the JWT payload set at login, which the server itself signed.
- Route composition is always `authenticate` → (`isAdmin` for admin-only
  routes) → controller. An admin route is never gated by `isAdmin` alone
  without `authenticate` first, since `isAdmin` depends on `req.user` already
  being set.

## Client-side role checks

- The client also checks the authenticated user's role (decoded from the
  stored token or returned at login) to hide admin-only UI — nav links,
  admin pages/routes, admin actions — from customers.
- This check exists purely for UX (not showing controls a customer can't
  use); it is not a security boundary. Hiding a button on the client never
  substitutes for the server-side `isAdmin` check — every admin endpoint
  enforces its own role check independent of what the client renders.
