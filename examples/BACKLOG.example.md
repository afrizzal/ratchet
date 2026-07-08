# BACKLOG — acme-todo API hardening
<!-- ratchet:v1 -->

> Example backlog for a fictional Express + TypeScript todo API. Shows every feature of the
> format: global checks, priorities, dependencies, human-gate tags, a ledger mid-run, and a
> journal spanning two sessions (i.e., a resume).

## Global checks
- `npx tsc --noEmit`
- `npm test`

## Items

### SEC-01 — Scope todo lookups by owner [P0]
- Tags: —
- Depends: —
- Evidence: src/routes/todos.ts:41 — GET/DELETE `/todos/:id` fetch by id only; any
  authenticated user can read or delete another user's todo
- Spec: add `ownerId: req.user.id` to both queries; switch the delete to `deleteMany` so the
  write is scoped too; return 404 (not 403) on miss to avoid existence leaks
- Acceptance:
  - [ ] `npm test` exits 0 (includes the new regression test below)
  - [ ] new test: user B requesting user A's todo id receives 404
  - [ ] existing todo CRUD tests still pass unchanged

### SEC-02 — Add regression test utilities for multi-user auth [P1]
- Tags: —
- Depends: SEC-01
- Evidence: test/helpers.ts:1 — helpers can only mint one test user; cross-user cases are
  currently untestable, which is how SEC-01 survived
- Spec: extend `makeTestUser()` to accept a seed suffix and export `makeTwoUsers()`; no
  changes to production code
- Acceptance:
  - [ ] `npm test` exits 0
  - [ ] `makeTwoUsers()` used by at least the SEC-01 regression test

### PERF-03 — Cap unbounded list endpoint [P2]
- Tags: —
- Depends: —
- Evidence: src/routes/todos.ts:12 — `GET /todos` returns every row; no limit parameter
- Spec: default `limit=50`, max 200, validated; add `?cursor` passthrough but do NOT build
  full pagination (that's a product decision — see OPS/product note in Journal)
- Acceptance:
  - [ ] `npx tsc --noEmit` exits 0
  - [ ] request without params returns ≤50 rows
  - [ ] `?limit=9999` is clamped to 200, not an error

### OPS-04 — Rotate the leaked staging DB password [P0]
- Tags: [OPS]
- Depends: —
- Evidence: scripts/seed-staging.sh:8 — plaintext password committed in git history
- Spec: rotate the credential on the staging server, move the script to read `PGPASSWORD`
  from env, then scrub or accept history exposure (owner's call)
- Acceptance:
  - [ ] old password no longer authenticates against staging
  - [ ] `grep -r "hunter2" scripts/` returns nothing

### UX-05 — Friendlier validation errors [P3]
- Tags: [USER-DECISION]
- Depends: —
- Evidence: src/middleware/validate.ts:22 — zod errors returned raw; users see internal paths
- Spec: needs a copy/product decision on error message wording before implementation
- Acceptance:
  - [ ] (to be defined together with the wording decision)

## Ledger
| ID | Status | Attempts | Commit | Note |
|---|---|---|---|---|
| SEC-01 | done | 2 | 3f9a2e1 | attempt 1 broke an unrelated fixture; see journal |
| SEC-02 | in-progress | 1 | — | resumed by session 2 |
| PERF-03 | todo | 0 | — | — |
| OPS-04 | needs-human | 0 | — | [OPS] requires staging server access |
| UX-05 | needs-human | 0 | — | [USER-DECISION] wording not decided |

## Journal
- 2026-07-06 run started (session 1). Baseline green: tsc 0, tests 34/34.
- 2026-07-06 SEC-01 attempt 1: scoping fix correct but `seed.ts` fixture assumed global
  visibility → 2 unrelated tests red. Reverted nothing (failure was in new test wiring),
  fixed the fixture within spec (it exists to serve these tests). Re-ran.
- 2026-07-06 SEC-01 attempt 2: green (35/35 incl. new regression test). Committed 3f9a2e1.
- 2026-07-06 OPS-04: human-gated [OPS], parked as needs-human. UX-05: [USER-DECISION], parked.
- 2026-07-06 session 1 ended (context budget). Resume: re-run /ratchet-loop on this file.
- 2026-07-07 run resumed (session 2). Ledger read: SEC-02 eligible (SEC-01 done). Marked
  in-progress. Suggestion for humans: consider a lint rule banning unscoped `findUnique` on
  owned models — recurring pattern behind SEC-01.
