# Git Conventions

## Commits

Commits follow [Conventional Commits](https://www.conventionalcommits.org/):
`<type>: <short imperative subject>`.

Types in use:

| Type | Use for |
| --- | --- |
| `feat` | A new feature or user-facing capability |
| `fix` | A bug fix |
| `docs` | Documentation only (e.g. files under `/docs`) |
| `refactor` | Restructuring code with no behavior change |
| `test` | Adding or changing tests |
| `chore` | Everything else that isn't user-facing (deps, config, tooling) |

The subject is short and imperative ("add cart quantity validation," not
"added" or "adds"), and describes what the commit does, not what was wrong
before it.

```
feat: add product variant stock editing to admin
fix: correct cart total rounding on discounted items
docs: document Razorpay webhook verification
refactor: extract order status transitions into a service
```

A commit's body (when needed) explains *why*, per the repo's general
guidance — not a restatement of the diff.

## Branches

Branches are named `<type>/<feature>`, using the same `type` prefixes as
commits, and a short kebab-case description of the feature:

```
feat/product-catalog
fix/cart-total
refactor/order-service
docs/payments-flow
```

## Workflow

- Each feature (or fix, refactor, etc.) is built on its own branch cut from
  `main` — work is never committed straight to `main`.
- Branches are merged into `main` through a pull request, never pushed or
  merged directly.
- PRs keep history clean: commits on the branch are meaningful and
  reviewable (not a pile of "wip"/"fix typo" commits left as-is), rebased or
  squashed as needed before merge so `main`'s history reads as a sequence of
  coherent, intentional changes.
- Every PR requires a **human review step before merge** — no PR merges
  purely on green CI without a reviewer's approval.
