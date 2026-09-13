---
name: plan-feature
description: Auto-trigger at the start of any new feature, before writing code — whenever the user starts planning, scoping, designing, or asking "how should we build/add/implement X" for a new capability in this app (e.g. "let's add wishlists," "plan out the reviews feature," "how should we implement coupon codes"). Enters Plan Mode, produces a short technical plan (files to create/change on server and client, referencing the relevant /docs), ends with a QA Scenarios section, waits for explicit approval, and on approval writes the plan to ./plans/<branch>.md. Do not use for small bug fixes, one-line tweaks, or purely exploratory/explain-only questions with no intent to build.
---

# Plan Feature

Produces a short, doc-grounded technical plan for a new feature before any
code is written, gets explicit approval, then persists the approved plan to
`./plans/<current-branch>.md`.

## When this runs

At the very start of scoping/planning a new feature — before touching any
source file. If code has already been written for the feature in this
session, this skill is the wrong tool; only use it for up-front planning.

## Steps

1. **Enter Plan Mode** (`EnterPlanMode`) before reading further into the
   codebase or writing anything. No file should be created or edited while
   in this skill until the plan is approved.

2. **Read the relevant docs first**, not from memory:
   - Always: `docs/architecture.md`, `docs/database.md`, `docs/api.md`.
   - Plus whichever of these match the feature area: `docs/auth.md`,
     `docs/admin.md`, `docs/payments.md`, `docs/security.md`,
     `docs/errors-and-validation.md`, `docs/ui.md`,
     `docs/coding-standards.md`, `docs/git-conventions.md`.
   - Skim the existing code the feature touches (relevant `server/src`
     layers, relevant `client/src` features/pages) so the plan matches what
     is actually there, not just what the docs describe in the abstract.
   - If a library/framework is involved (Express, Mongoose, React,
     Razorpay, etc.), pull current docs via Context7 before proposing any
     API usage, per CLAUDE.md.

3. **Draft the plan.** Keep it short and concrete, not exhaustive prose.
   Structure:

   - **Summary** — one or two sentences on what the feature does.
   - **Docs consulted** — bullet list of the `/docs` files referenced, with
     the specific section/pattern each one drives (e.g. "database.md —
     Order schema access patterns," "admin.md — role-gating pattern").
   - **Server changes** — bullet list of files to create/change, one line
     each, organized by layer (`routes` → `middleware` → `controllers` →
     `services` → `models`) per architecture.md's request flow. Note new
     Mongoose fields/indexes, new Zod schemas (and that they live in
     `shared`), and which routes are customer-scoped (`user: req.user.id`)
     vs. admin-gated (`isAdmin`, unscoped) per the authorization pattern in
     CLAUDE.md/admin.md — never mix the two.
   - **Client changes** — bullet list of files to create/change, one line
     each, organized by `components/ui` → `features/[feature]` → `pages`
     composition direction. Note new shared types/schemas consumed from
     `@kicks/shared`.
   - **Shared changes** — new/changed types or Zod schemas in `shared`, if
     any, and that `pnpm build:shared` is needed before client/server pick
     them up.
   - **Open questions** — anything genuinely ambiguous that affects the
     plan, if any.
   - **QA Scenarios** — 3 to 6 concrete scenarios, each one line stating
     what the user does and what should happen. Cover, as applicable to the
     feature:
     - Happy path
     - Auth boundary (unauthenticated request is rejected)
     - Role boundary (customer vs. admin — a customer route never returns
       another user's data; an admin-only action is rejected for a
       customer)
     - Validation (a malformed/missing input is rejected with a clear
       error, per errors-and-validation.md)
     - A relevant edge case (e.g. out-of-stock variant, cancelled order,
       already-paid order — whatever is specific to this feature)

4. **Present the plan and wait for explicit approval.** Use
   `ExitPlanMode` to hand the plan to the user for approval. Do not create
   or edit any source file until the user has explicitly approved the plan
   in this turn — a request for changes is not approval; revise and
   re-present.

5. **On approval, persist the plan:**
   - Run `git branch --show-current` to get the current branch name.
   - Write the approved plan's full content to `./plans/<branch>.md`
     (create the `plans/` directory if it doesn't exist). Use the branch
     name verbatim as the filename (sanitize only characters illegal in a
     filename, e.g. `/` → `-`).
   - Confirm the file path to the user, then proceed to implementation.
