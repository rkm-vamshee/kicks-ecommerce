# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Kicks — a MERN e-commerce app written in TypeScript, structured as a pnpm
workspace with three packages: `client` (React + Vite), `server` (Express +
Node), and `shared` (types/Zod schemas used by both). See
[docs/architecture.md](docs/architecture.md), [docs/database.md](docs/database.md),
and [docs/ui.md](docs/ui.md) for the full design docs — read them before
making structural changes, they're more detailed than this file.

## Commands

Run from the repo root with pnpm (`pnpm@11.20.0`, Node >=20):

```
pnpm dev:server         # tsx watch server/src/index.ts
pnpm dev:client         # vite dev server for client
pnpm build:shared       # tsc build for shared (build this first — client/server depend on it)
pnpm build:server
pnpm build:client
pnpm lint               # lints server, client, shared in that order
pnpm format             # prettier --write .
pnpm format:check
```

Per-package commands (`pnpm --filter <server|client|shared> <script>`):
- `typecheck` — `tsc --noEmit` (or `tsc -b --noEmit` for client)
- `lint` — `eslint .`

There is no test runner configured in any package yet — don't invent test
commands or assume a framework (Jest/Vitest/etc.) is present.

`shared` must be built (`pnpm build:shared`) before `client` or `server` will
pick up changes to its exported types/schemas, since both consume it as a
workspace package (`@kicks/shared`) resolved to `shared/dist`, not to
`shared/src` directly.

## Architecture

Request/data flow: `client` talks to `server` only over the REST API (never
touches MongoDB directly); `server` owns all business logic and is the only
part of the system with database access; `shared` holds TypeScript types and
Zod schemas imported by both sides so request/response shapes can't drift
apart.

### Server (`/server/src`)

Layered request flow: `routes` → `middleware` → `controllers` → `services` → `models`.

| Directory | Responsibility |
| --- | --- |
| `routes` | Express route definitions; map HTTP verb + path to a controller. |
| `middleware` | Cross-cutting concerns: auth, validation, error handling. |
| `controllers` | Parse request, call services, shape the response. |
| `services` | Business logic, reusable across controllers. |
| `models` | Mongoose schemas/models. |

Database: a single cached Mongoose connection created once at startup
(`connectDB()` in `server/config/db.ts`), not opened per-request. Every
schema uses `strict: true`, `timestamps: true`, and declares field-level
validation and indexes on the fields actually queried — see
[docs/database.md](docs/database.md) for the concrete `User`/`Product`/`Cart`/`Order`
shapes before adding or changing a model.

**Authorization pattern** (enforced in `services`/`middleware`, not just
routes): customer-facing queries on `Cart`/`Order` always filter by
`user: req.user.id` — ownership is baked into the query. Admin routes are
gated by an `isAdmin` middleware checking `req.user.role === 'admin'` before
the controller runs, and once past that gate they query unscoped (no `user`
filter). Never mix the two: an admin route must not filter by `user`, and a
customer route must never skip the `user` filter.

### Client (`/client/src`)

| Directory | Responsibility |
| --- | --- |
| `pages` | Top-level route/page components; composes feature components, handles routing/data-fetching. |
| `features/[feature]` | Feature-scoped components/hooks/logic grouped by domain (`cart`, `checkout`, `product`, ...). |
| `components/ui` | Reusable, feature-agnostic shadcn/ui primitives. |

Composition direction is one-way: `components/ui` primitives → `features/*`
domain components (compose primitives + domain data, never reimplement a
primitive) → `pages/*` (compose feature components). shadcn primitives are
generated into `components/ui` via `pnpm dlx shadcn@latest add <component>`
(config in `client/components.json`) and are treated as project code once
generated — safe to edit locally, not upstreamed.

Styling: Tailwind + shadcn (`slate` base theme, CSS variables in
`client/src/index.css`, radius/color tokens documented in
[docs/ui.md](docs/ui.md)). Visual direction takes cues from Shopify
storefronts — product-first, whitespace-heavy layouts.

### Naming conventions

- Files: kebab-case (`product-card.tsx`, `order-service.ts`).
- Components and types: PascalCase (`ProductCard`, `OrderSummary`).

## Environment

Server env vars (`server/.env.example`): `PORT`, `NODE_ENV`, `MONGO_URI`,
`JWT_SECRET`, `JWT_EXPIRES_IN`, `CLIENT_URL` (used for CORS origin).
Client env vars (`client/.env.example`): `VITE_API_URL`.
