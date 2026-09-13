# Architecture

Kicks is a MERN application written in TypeScript, structured as a single
repository split into a `client`, a `server`, and a `shared` package.

## Overview

- **`/client`** — React + Vite single-page app. It talks to the server only
  through the REST API and never touches the database directly.
- **`/server`** — Express + Node REST API. It owns all business logic and is
  the only part of the system with database access.
- **`/shared`** — TypeScript types and Zod schemas consumed by both `client`
  and `server`, so request/response shapes stay in sync across the API
  boundary.

```
client  <-- REST API -->  server  <-->  database
   \                         /
    \---------------------/
         shared types
```

## Server layout (`/server`)

| Directory              | Responsibility                                                   |
| ----------------------- | ----------------------------------------------------------------- |
| `/server/routes`        | Express route definitions; map HTTP verbs + paths to controllers. |
| `/server/controllers`   | Request/response handling; parse input, call services, shape output. |
| `/server/models`        | Database schema/models (Mongoose).                                |
| `/server/middleware`    | Cross-cutting request handling (auth, validation, error handling). |
| `/server/services`      | Business logic, reusable across controllers.                      |

Request flow: `routes` → `middleware` → `controllers` → `services` → `models`.

## Client layout (`/client/src`)

| Directory                          | Responsibility                                            |
| ------------------------------------ | ----------------------------------------------------------- |
| `/client/src/pages`                  | Top-level route/page components.                            |
| `/client/src/features/[feature]`     | Feature-scoped components, hooks, and logic grouped by domain (e.g. `cart`, `checkout`). |
| `/client/src/components/ui`          | Reusable, feature-agnostic UI primitives (shadcn components). |

## Naming conventions

- **Files**: kebab-case (e.g. `product-card.tsx`, `order-service.ts`).
- **Components and types**: PascalCase (e.g. `ProductCard`, `OrderSummary`).

## Boundaries

- The client never queries the database directly — all data access goes
  through the server's REST API.
- The server owns all business logic; the client is presentation-only and
  should not duplicate domain rules.
- Shared request/response types and validation schemas live in `/shared` and
  are imported by both `client` and `server` rather than redefined on each
  side.
