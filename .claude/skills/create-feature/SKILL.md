---
name: create-feature
description: Auto-trigger when starting to build/implement/code a feature that already has an approved plan — e.g. "let's build it," "implement the plan," "start building the wishlist feature," "go ahead and code this up." Reads the approved plan in ./plans/<branch>.md plus the relevant /docs files, pulls current library docs via Context7, and writes the feature's server/client/shared code following project conventions. Writes code only — no tests, no QA, no commits. Do not use for planning/scoping (use plan-feature first), for writing tests (use write-tests), for running QA (use run-qa-suite), or for small one-off fixes with no plan behind them.
---

# Create Feature

Implements an already-approved feature plan. This skill only writes
production code — it never writes tests, runs QA, or commits.

## When this runs

After a feature plan has been approved (typically via the `plan-feature`
skill). If there is no approved plan for the current work, stop and ask for
one (or run `plan-feature` first) rather than improvising scope.

## Steps

1. **Load the plan.** Run `git branch --show-current` and read
   `./plans/<branch>.md`. If it doesn't exist, ask the user for the plan (or
   suggest running `plan-feature`) before writing any code — don't guess at
   scope.

2. **Read the relevant docs**, not from memory — the same set the plan was
   built against:
   - Always: `docs/architecture.md`, `docs/database.md`, `docs/api.md`.
   - Plus whichever apply to this feature: `docs/auth.md`, `docs/admin.md`,
     `docs/payments.md`, `docs/security.md`,
     `docs/errors-and-validation.md`, `docs/ui.md`,
     `docs/coding-standards.md`.
   - Check the plan's "Docs consulted" section for the specific
     sections/patterns it called out and re-confirm them against the actual
     doc content before implementing.

3. **Pull live library docs via Context7** before writing code that uses
   Express, Mongoose, React, Razorpay, or any other external
   library/framework/API — per CLAUDE.md, never rely on training data for
   API signatures or version-specific behavior. If Context7 has no entry for
   a library, say so explicitly before falling back to memorized knowledge.

4. **Implement exactly what the plan describes**, following project
   conventions:
   - Server: `routes` → `middleware` → `controllers` → `services` →
     `models`, per architecture.md. Customer routes filter by
     `user: req.user.id`; admin routes gate on `isAdmin` and query
     unscoped — never mix the two.
   - Shared: new/changed types and Zod schemas go in `shared` and are
     imported by both `client` and `server`. Run `pnpm build:shared` after
     changing it, before relying on the update from `client` or `server`.
   - Client: `components/ui` primitives → `features/[feature]` domain
     components → `pages`, one-way composition.
   - Naming: kebab-case files, PascalCase components/types.
   - Validation lives in Zod schemas at the route boundary
     (`validate(schema)` middleware), not inside controllers.
   - If implementation reveals the plan is wrong, incomplete, or
     inconsistent with the docs, stop and flag the discrepancy to the user
     rather than silently deviating from the approved plan.

5. **Do not**:
   - Write or update tests.
   - Run lint/typecheck/build as a QA pass, or otherwise attempt to
     validate the feature end-to-end (that's `run-qa-suite`).
   - `git add` or `git commit` anything.
   - Expand scope beyond what the plan describes.

6. **When the build is done, stop.** Report in 2-3 lines what was built
   (files created/changed, by area), then hand back. Mention that
   `write-tests` and `run-qa-suite` are the next steps, without running them.
