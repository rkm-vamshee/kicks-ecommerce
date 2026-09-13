# UI

Kicks' client is built with [shadcn/ui](https://ui.shadcn.com) on top of
Tailwind CSS. Visually it takes cues from Shopify storefronts: clean,
product-first layouts, generous whitespace, and typography/imagery doing the
work rather than heavy chrome or decoration.

## Component conventions

### Where shadcn primitives live

shadcn is configured via [`client/components.json`](../client/components.json):

- Style: `default`, base color `slate`, CSS variables on (not utility classes
  baked into each component).
- Generated primitives are installed into `client/src/components/ui` (the
  `ui` alias) — e.g. `button.tsx`, `card.tsx`, `dialog.tsx`, `input.tsx`.
- These files are copied into the repo by the shadcn CLI (`pnpm dlx
  shadcn@latest add <component>`), not imported from a package. Once
  generated, they're treated as **our code** — safe to edit directly for
  project-specific tweaks (sizing, variants), but changes stay local rather
  than being upstreamed.
- `client/src/components/ui` only ever holds feature-agnostic primitives. A
  component belongs here if it has no knowledge of products, carts, orders,
  or any other domain concept — buttons, inputs, dialogs, dropdowns, cards as
  layout shells, etc.

### How feature components compose primitives

- Domain components live under `client/src/features/[feature]` (see
  [architecture.md](./architecture.md)), one directory per domain: `product`,
  `cart`, `checkout`, `order`, etc.
- A feature component composes one or more `ui` primitives plus domain data
  and never re-implements what a primitive already does. E.g. `ProductCard`
  wraps `Card` + `Button`; it doesn't hand-roll its own card shell.
- Feature components are named for the domain concept they render
  (`ProductCard`, `CartDrawer`, `OrderSummary`), following the PascalCase /
  kebab-case file naming from [architecture.md](./architecture.md).
- Page components (`client/src/pages`) compose feature components; they hold
  routing/data-fetching concerns and layout, not primitive-level markup.

```
components/ui/*        <- shadcn primitives (Button, Card, Dialog, Input, ...)
       ^
features/product/*     <- ProductCard, ProductGallery, PriceTag (compose primitives)
       ^
pages/*                 <- ProductDetailPage (composes feature components)
```

## Layout patterns

### Storefront

- **Product grid** — Responsive CSS grid, product-image-forward cards
  (image, name, price; rating/badges optional), generous gutter between
  cards so products don't compete visually. Column count scales with
  breakpoint (see [Breakpoints](#breakpoints)). Exact column counts per
  breakpoint: **TBD**.
- **Product detail** — Two-column layout above the fold on desktop (image
  gallery left, product info + add-to-cart right), collapsing to a single
  stacked column on mobile with the gallery first. Detail sections below the
  fold (description, specs, reviews) as full-width stacked blocks.
- **Cart drawer** — Slide-in panel (shadcn `Sheet`), anchored right, opened
  from the header cart icon rather than navigating to a separate page for
  quick edits. Line items list + subtotal + checkout CTA pinned to the
  bottom of the drawer.
- **Checkout** — Single-column, linear flow (shipping → payment → review),
  order summary visible alongside (desktop) or collapsible above the form
  (mobile). Exact step structure and whether it's one page or multi-step:
  **TBD**.

### Admin area

- **Data tables** — shadcn `Table` composed with `@tanstack/react-table` for
  sorting/filtering/pagination (products, orders, users lists). Row actions
  (edit/delete) as an overflow menu (`DropdownMenu`) in the last column.
- **Forms** — shadcn `Form` (wrapping `react-hook-form`) + Zod resolver,
  reusing the validation schemas from `/shared` so admin forms validate
  against the same rules as the API. Field-level errors inline under each
  input.
- **Stat cards** — `Card` primitive as the shell, a small grid of them at
  the top of dashboard-style pages (e.g. orders today, revenue, low stock).
  Content: label, value, optional delta/trend — no charting library chosen
  yet: **TBD**.

## Design tokens

These are the actual Tailwind/shadcn tokens currently configured in
[`client/tailwind.config.js`](../client/tailwind.config.js) and
[`client/src/index.css`](../client/src/index.css). Colors are HSL CSS
variables (shadcn default `slate` theme) so light/dark mode is a variable
swap, not a separate component tree.

### Type scale

Not yet customized — Tailwind's default type scale (`text-xs` through
`text-9xl`) is in use as-is, no project-specific scale defined in
`tailwind.config.js`. A deliberate scale for product names, prices, and
headings: **TBD**.

### Spacing

Not yet customized — Tailwind's default spacing scale. The `container`
utility is configured with `center: true`, `padding: '2rem'`, and a `2xl`
breakpoint capped at `1400px`, so page content is centered with a 2rem
gutter at every breakpoint up to that cap. A denser/looser spacing scale for
product grids specifically: **TBD**.

### Radius

Single source of truth: `--radius: 0.5rem` (`src/index.css`), consumed by
Tailwind as three steps:

| Token       | Value                    |
| ----------- | ------------------------ |
| `rounded-lg`| `var(--radius)` (`0.5rem`) |
| `rounded-md`| `calc(var(--radius) - 2px)` |
| `rounded-sm`| `calc(var(--radius) - 4px)` |

shadcn primitives use these tokens rather than hardcoded radius values, so
changing `--radius` re-skins every component at once.

### Breakpoints

Tailwind defaults (unmodified): `sm` 640px, `md` 768px, `lg` 1024px, `xl`
1280px, `2xl` 1536px (container caps at `1400px` per above). No custom
breakpoints defined yet.

### Colors

| Token         | Light                | Usage                              |
| -------------- | --------------------- | ------------------------------------ |
| `background`  | `0 0% 100%`          | Page background |
| `foreground`  | `240 10% 3.9%`       | Default text |
| `card`        | `0 0% 100%`          | Card surfaces |
| `primary`     | `240 5.9% 10%`       | Primary actions (buttons, links) |
| `secondary`   | `240 4.8% 95.9%`     | Secondary surfaces/actions |
| `muted`       | `240 4.8% 95.9%`     | Subdued backgrounds, disabled state |
| `accent`      | `240 4.8% 95.9%`     | Hover/highlight state |
| `destructive` | `0 84.2% 60.2%`      | Delete/cancel/error actions |
| `border`      | `240 5.9% 90%`       | Borders, dividers |
| `ring`        | `240 5.9% 10%`       | Focus rings |

Dark mode variants are defined under `.dark` in `index.css` — same token
names, inverted values. A brand accent color distinct from the default slate
theme (e.g. for sale badges, promotional callouts) is not yet chosen: **TBD**.
