# BACKLOG — acme-todo API hardening
<!-- ratchet:v2 -->

> Example **v2** backlog for the same fictional Express + TypeScript todo API as
> `examples/BACKLOG.example.md` (v1), migrated to parallel lanes. State lives in
> `lanes/*.md`, one file per lane, one writer each — so an `api` executor and a `ui`
> executor can run at the same time in separate worktrees without ever writing the
> same file. This file is **human-owned**: executors read it and never write it.
>
> Start with the [walkthrough](README.md) — it shows two agents working this backlog at once.
> Companion files: `NEXT.example.md`, `lanes/core.example.md`, `lanes/api.example.md`,
> `lanes/ui.example.md`.

## Global checks
- `npx tsc --noEmit --incremental false`
- `npm test`

## Lanes
| Lane | Scope | Notes |
|---|---|---|
| core | (rest) | default lane — dependencies, config, integration fixes; barrier runs only |
| api | `src/routes/**`, `src/middleware/**`, `test/**` | server hardening |
| ui | `web/src/**` | frontend debt |

## Items

### SEC-01 — Scope todo lookups by owner [P0]
- Lane: api
- Tags: —
- Depends: —
- Evidence: src/routes/todos.ts:41 — GET/DELETE `/todos/:id` fetch by id only; any
  authenticated user can read or delete another user's todo
- Spec: add `ownerId: req.user.id` to both queries; switch the delete to `deleteMany` so the
  write is scoped too; return 404 (not 403) on miss to avoid existence leaks; add a regression
  test (user B requesting user A's todo id → 404) — mint the second test user inline for now,
  SEC-02 extracts the shared helper afterwards
- Acceptance:
  - [ ] `npm test` exits 0 (includes the new regression test below)
  - [ ] new test: user B requesting user A's todo id receives 404
  - [ ] existing todo CRUD tests still pass unchanged

### SEC-02 — Add regression test utilities for multi-user auth [P1]
- Lane: api
- Tags: —
- Depends: SEC-01
- Evidence: test/helpers.ts:1 — helpers can only mint one test user; cross-user auth/ownership
  cases are currently hard to write, which is how SEC-01 survived
- Spec: in test/helpers.ts, extend `makeTestUser()` to accept a seed suffix and export
  `makeTwoUsers()`; no changes to production code
- Acceptance:
  - [ ] `npm test` exits 0
  - [ ] `makeTwoUsers()` used by at least the SEC-01 regression test

### PERF-03 — Cap unbounded list endpoint [P2]
- Lane: api
- Tags: —
- Depends: —
- Evidence: src/routes/todos.ts:12 — `GET /todos` returns every row; no limit parameter
- Spec: default `limit=50`, max 200, validated; add `?cursor` passthrough but do NOT build
  full pagination (that's a product decision — see the 2026-07-06 PERF-03 note in the Journal)
- Acceptance:
  - [ ] `npx tsc --noEmit --incremental false` exits 0
  - [ ] `curl -s localhost:3000/todos | jq length` returns ≤ 50 against the 60-row seed DB
  - [ ] `curl -s 'localhost:3000/todos?limit=9999' | jq length` returns ≤ 200 with HTTP 200 (clamped, not an error)

### UI-06 — Virtualize the todo list [P2]
- Lane: core
- Tags: —
- Depends: —
- Evidence: web/src/TodoList.tsx:18 — renders every row; 2k-todo accounts drop frames on scroll
- Spec: render rows through `@tanstack/react-virtual` (new dependency — this is why the item
  is in the `core` lane: it edits package.json + the lockfile, which no named lane owns);
  keep the row markup and props identical
- Acceptance:
  - [ ] `npx tsc --noEmit --incremental false` exits 0
  - [ ] `npm test` exits 0 (the TodoList snapshot test still passes)

### UI-07 — Empty state for a filtered-to-nothing list [P3]
- Lane: ui
- Tags: —
- Depends: —
- Evidence: web/src/TodoList.tsx:44 — an active filter with zero matches renders a blank pane
- Spec: when `todos.length === 0 && hasActiveFilter`, render the existing `<EmptyState>`
  component with copy "No todos match this filter"; do not touch the unfiltered empty case
- Acceptance:
  - [ ] `npm test` exits 0 (add a render test asserting the copy appears)

### OPS-04 — Rotate the leaked staging DB password [P0]
- Lane: core
- Tags: [OPS]
- Depends: —
- Evidence: scripts/seed-staging.sh:8 — plaintext password committed in git history
- Spec: rotate the credential on the staging server, move the script to read `PGPASSWORD`
  from env, then scrub or accept history exposure (owner's call)
- Acceptance:
  - [ ] old password no longer authenticates against staging
  - [ ] `grep -r "hunter2" scripts/` returns nothing

### UX-05 — Friendlier validation errors [P3]
- Lane: api
- Tags: [USER-DECISION]
- Depends: —
- Evidence: src/middleware/validate.ts:22 — zod errors returned raw; users see internal paths
- Spec: needs a copy/product decision on error message wording before implementation
- Acceptance:
  - [ ] (provisional — gated on the wording decision; replace with a real criterion at unpark)
