# Admin

The admin area is a role-gated part of the app for managing products,
orders, and store stats. It is gated by the `admin` role on **both** the
server and the client — but, as with all authorization in Kicks (see
[auth.md](./auth.md)), only the server-side check is a security boundary.

## What admins can do

- **Manage products**: create, edit, and delete products, and set stock
  **per size variant** (not a single stock number for the whole product —
  see the `variants` field on `Product` in [database.md](./database.md#product),
  where each variant embeds its own `size` and `stock`). Editing stock for
  one size never touches another size's stock on the same product.
- **View and update all orders**: list every customer's orders (not scoped
  to `req.user.id` the way a customer's own order list is — see the
  admin-vs-customer query pattern in
  [database.md](./database.md#access-patterns)), change an order's `status`
  (per the state flow in [payments.md](./payments.md)), and mark a COD
  order's `paymentStatus` as paid after delivery (per
  [payments.md](./payments.md#cash-on-delivery-cod)).
- **See basic store stats**: a dashboard-style view (stat cards, per
  [ui.md](./ui.md#admin-area)) — e.g. orders today, revenue, low-stock
  variants. Exact metrics and their source queries: **TBD**.

## What admins never see

- **Customer passwords** — only ever exist as a bcrypt hash (per
  [auth.md](./auth.md#password-storage)); no admin view, API response, or
  export ever includes a password field, hashed or otherwise.
- **Payment card data** — Razorpay's hosted checkout handles card details
  directly with the customer (per [payments.md](./payments.md#razorpay));
  Kicks's server and database never receive or store raw card numbers,
  CVVs, or expiry dates, so there is nothing card-related for an admin view
  to display in the first place.

Admin views of orders/customers show only what the store itself needs to
operate — order contents, shipping address, order/payment status — never
credentials or raw payment instrument data.

## Client: `/admin` route group

- Admin UI lives under an `/admin` route group in the client (`client/src/pages/admin/...`
  or equivalent), separate from the customer-facing storefront routes.
- The route group is gated by a client-side role check (the authenticated
  user's `role`, per [auth.md](./auth.md#client-side-role-checks)): a
  non-admin visiting an `/admin` route is redirected away rather than shown
  the page. This includes hiding admin nav entry points (links, menu items)
  from non-admin users, not just guarding the route itself.
- This client-side gate is **UX only** — it exists so customers never see
  admin screens they have no use for, not to keep out someone who
  deliberately navigates to `/admin` or calls the API directly. It is not
  the thing that makes admin actions safe.

## Server: every admin action is authorized in middleware

- Every admin-only API route (`POST/PATCH/DELETE /api/products`, `GET
  /api/orders` unscoped, `PATCH /api/orders/:id`, stats endpoints) is gated
  by `authenticate` → `isAdmin` middleware on the server, per
  [auth.md](./auth.md#role-based-access) and
  [api.md](./api.md#auth-requirements-per-route-group). This check runs
  independent of, and regardless of, whatever the client renders.
- The client's role check and the server's `isAdmin` middleware are not two
  halves of one check — the server check alone is sufficient and mandatory
  for every admin action; the client check is a convenience layered on top
  of it, never a substitute for it, and its absence or bypass (e.g. a direct
  API call, a modified client bundle) must never grant access the server
  wouldn't otherwise allow.
