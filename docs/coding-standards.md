# Coding Standards

## TypeScript

- Strict mode is on across `server` and `shared` (`"strict": true` in their
  `tsconfig.json`). **Note:** `client/tsconfig.app.json` and
  `tsconfig.node.json` do not currently set `"strict": true` — this is a gap
  against the standard, not an intentional exception; it should be added
  rather than treated as how client code is allowed to diverge.
- No `any`. If a value's type genuinely can't be known, use `unknown` and
  narrow it before use — `any` defeats the type checker rather than working
  within it.
- No `@ts-ignore` (or `@ts-expect-error`) without a comment on the same line
  explaining *why* the suppression is needed. A suppression with no
  explanation is treated the same as `any` — not allowed.

## Formatting and linting

- Prettier formats all code (`pnpm format` / `pnpm format:check` at the
  root); config is in `.prettierrc.json`. Don't hand-format to a different
  style or argue with Prettier's output in review — fix it by running
  Prettier, not by manually matching it.
- ESLint lints each package (`pnpm lint`, or `pnpm --filter <pkg> lint`),
  built on `typescript-eslint`'s recommended rules plus
  `eslint-plugin-react-hooks` / `eslint-plugin-react-refresh` for `client`.
  Lint runs clean before code is considered done — don't disable a rule
  inline to make a warning go away without understanding why it fired.

## Imports

Imports are grouped, in this order, with internal ordering by path/name
within each group:

1. External packages (`react`, `express`, `zod`, `@kicks/shared`, ...)
2. Internal aliases (`@/components/...`, `@/lib/...` in `client`)
3. Relative imports (`./`, `../`)

```ts
import { useState } from 'react';
import { z } from 'zod';

import { Button } from '@/components/ui/button';
import { useCart } from '@/hooks/use-cart';

import { ProductPrice } from './product-price';
```

`@kicks/shared` counts as an external package for ordering purposes (it's a
workspace dependency resolved like any other package import), not an
internal alias.

## Functions

- Functions are declarations (`function getProduct(id: string): Promise<Product>
  { ... }`), not anonymous arrow functions assigned to a `const`, except
  where the language requires an expression (a callback passed inline, a
  React component using a specific pattern already established in that
  file). A named function declaration is preferred wherever a top-level or
  exported function is being defined.
- Parameters and return types are explicitly typed. Inference is fine for
  local variables inside a function body, but a function's public
  signature — what it takes and what it returns — is never left for callers
  to infer from the implementation.

## Async

- Async work uses `async`/`await`. `.then()`/`.catch()` chains are not
  used — an async function that needs to handle a rejection wraps the
  `await` in `try`/`catch` instead of chaining `.catch()`.

## Errors

- Errors are thrown as real `Error` objects (or a subclass — e.g. a
  `NotFoundError` in a service, per [api.md](./api.md)), never as strings or
  plain objects (`throw new Error('...')`, never `throw '...'`). This keeps
  a stack trace attached to every thrown error and lets `catch` blocks and
  the centralized error-handling middleware rely on `instanceof Error`.

## Shared types

- A type or Zod schema that describes a shape crossing the client/server
  boundary (a request body, a response payload, a domain entity like
  `Product` or `Order`) is defined exactly once, in `/shared`, and imported
  by both `client` and `server` as `@kicks/shared`. It is never redefined or
  hand-copied in `client` or `server` — if the two sides' types drift, that's
  a sign the type should have lived in `shared` and didn't.
- Types that are purely internal to one side (a component's props, a
  service's internal parameter object) stay local to that package — `shared`
  holds only what's actually shared, not a dumping ground for every type in
  the app.
