# Database

Kicks uses MongoDB with Mongoose. Mongoose gives us schema definition,
validation, and a connection pool, so the server never talks to the MongoDB
driver directly — every read and write goes through a Mongoose model.

## Connection

The server keeps a single cached connection, created once and reused across
the app rather than opened per-request:

```ts
// server/config/db.ts
import mongoose from "mongoose";

let connection: typeof mongoose | null = null;

export async function connectDB() {
  if (connection) return connection;
  connection = await mongoose.connect(process.env.MONGODB_URI!);
  return connection;
}
```

`connectDB()` is called once at server startup (before the HTTP listener
starts accepting requests). Mongoose's driver manages pooling internally, so
route handlers and services never open or close connections themselves —
they just import a model and query it.

## Schema conventions

Every schema in `/server/models`:

- Is declared with `strict: true` (Mongoose's default) — fields not defined
  in the schema are silently dropped rather than persisted, so typos in a
  request body can never leak an unexpected field into the database.
- Has `timestamps: true`, giving every document `createdAt` and `updatedAt`.
- Declares field-level validation (`required`, `enum`, `min`/`max`, custom
  validators) in the schema itself, so invalid data is rejected at the model
  layer regardless of which service or controller writes it.
- Defines indexes on the fields that are actually queried on, not
  speculatively on every field.

## Core models

### User

| Field       | Type                          | Notes                          |
| ----------- | ------------------------------ | ------------------------------- |
| `email`     | `String`, required, unique     | Indexed — looked up on every login and auth check. |
| `password`  | `String`, required             | Stored as a bcrypt hash, never plaintext. |
| `name`      | `String`, required             | |
| `role`      | `String`, enum `["customer", "admin"]`, default `"customer"` | Drives authorization — see [Access patterns](#access-patterns). |

```ts
const userSchema = new Schema(
  {
    email: { type: String, required: true, unique: true, lowercase: true, trim: true },
    password: { type: String, required: true },
    name: { type: String, required: true },
    role: { type: String, enum: ["customer", "admin"], default: "customer" },
  },
  { strict: true, timestamps: true }
);

userSchema.index({ email: 1 });
```

### Product

| Field         | Type                          | Notes                          |
| ------------- | ------------------------------ | ------------------------------- |
| `name`        | `String`, required             | |
| `slug`        | `String`, required, unique     | Indexed — the primary lookup key for product pages. |
| `description` | `String`                       | |
| `price`       | `Number`, required, min `0`    | |
| `variants`    | `[VariantSchema]`              | Embedded subdocuments; a product has no meaning without its variants, and they're always read/written together with it. |

Each variant embeds its own size and stock:

```ts
const variantSchema = new Schema(
  {
    size: { type: String, required: true },
    stock: { type: Number, required: true, min: 0, default: 0 },
  },
  { _id: true }
);

const productSchema = new Schema(
  {
    name: { type: String, required: true },
    slug: { type: String, required: true, unique: true, lowercase: true, trim: true },
    description: String,
    price: { type: Number, required: true, min: 0 },
    variants: [variantSchema],
  },
  { strict: true, timestamps: true }
);

productSchema.index({ slug: 1 });
```

Stock is decremented on a specific variant (matched by its subdocument `_id`
or by `size`), never on the product as a whole.

### Cart

| Field       | Type                                              | Notes                          |
| ----------- | --------------------------------------------------- | -------------------------------- |
| `user`      | `ObjectId`, ref `User`, required, unique            | One active cart per user. |
| `items`     | `[{ product: ObjectId ref Product, variantId: ObjectId, quantity: Number }]` | Embedded — a cart's line items have no independent lifecycle. |

```ts
const cartItemSchema = new Schema(
  {
    product: { type: Schema.Types.ObjectId, ref: "Product", required: true },
    variantId: { type: Schema.Types.ObjectId, required: true },
    quantity: { type: Number, required: true, min: 1 },
  },
  { _id: false }
);

const cartSchema = new Schema(
  {
    user: { type: Schema.Types.ObjectId, ref: "User", required: true, unique: true },
    items: [cartItemSchema],
  },
  { strict: true, timestamps: true }
);
```

The unique index on `user` is a byproduct of `unique: true` above and doubles
as the lookup path — a customer's cart is always fetched by their own `user`
id, never scanned for.

### Order

| Field       | Type                                                          | Notes                          |
| ----------- | ---------------------------------------------------------------- | -------------------------------- |
| `user`      | `ObjectId`, ref `User`, required                                  | Indexed together with `status`. |
| `items`     | `[{ product: ObjectId ref Product, variantId: ObjectId, quantity: Number, price: Number }]` | Price is snapshotted at order time so later product price changes don't rewrite history. |
| `status`    | `String`, enum `["pending", "paid", "shipped", "delivered", "cancelled"]`, default `"pending"` | Indexed together with `user`. |
| `total`     | `Number`, required, min `0`                                       | |
| `shippingAddress` | `Object`                                                    | |

```ts
const orderItemSchema = new Schema(
  {
    product: { type: Schema.Types.ObjectId, ref: "Product", required: true },
    variantId: { type: Schema.Types.ObjectId, required: true },
    quantity: { type: Number, required: true, min: 1 },
    price: { type: Number, required: true, min: 0 },
  },
  { _id: false }
);

const orderSchema = new Schema(
  {
    user: { type: Schema.Types.ObjectId, ref: "User", required: true },
    items: [orderItemSchema],
    status: {
      type: String,
      enum: ["pending", "paid", "shipped", "delivered", "cancelled"],
      default: "pending",
    },
    total: { type: Number, required: true, min: 0 },
    shippingAddress: { type: Object },
  },
  { strict: true, timestamps: true }
);

orderSchema.index({ user: 1, status: 1 });
```

The compound index matches the two ways orders are actually queried: "all of
this user's orders" and "this user's orders in a given status" (e.g. their
pending order), as well as admin queries filtered by `status` alone.

## Access patterns

Authorization for these models is enforced in `/server/services` and
`/server/middleware`, not just at the route layer, and follows one rule
consistently:

- **Customer queries are scoped by ownership.** Any query a customer-facing
  controller runs against `Cart` or `Order` includes `user: req.user.id` as a
  filter. A customer can only ever read or mutate their own cart and their
  own orders — there is no code path where a customer's id is taken from the
  request body or params instead of the authenticated session, and no query
  fetches a cart or order by its own `_id` alone without also checking
  `user`.
- **Admin queries are scoped by role, not ownership.** Admin routes (listing
  all orders, updating order status, managing products) are gated by an
  `isAdmin` middleware that checks `req.user.role === "admin"` before the
  controller runs. Once past that gate, admin queries are free to read and
  write across all users' data — they don't filter by `user`, because the
  authorization already happened at the role check, not at the row.

In short: a customer's authorization is baked into the query (`WHERE user =
me`); an admin's authorization is a gate in front of the query (`IF role =
admin THEN run unscoped query`). The two should never be mixed — an admin
route must not filter by `user`, and a customer route must never skip the
`user` filter because "the role check happened already."
