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
  write is scoped too; return 404 (not 403) on miss to avoid existence leaks; add a regression
  test (user B requesting user A's todo id → 404) — mint the second test user inline for now,
  SEC-02 extracts the shared helper afterwards
- Acceptance:
  - [ ] `npm test` exits 0 (includes the new regression test below)
  - [ ] new test: user B requesting user A's todo id receives 404
  - [ ] existing todo CRUD tests still pass unchanged

### SEC-02 — Add regression test utilities for multi-user auth [P1]
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
- Tags: —
- Depends: —
- Evidence: src/routes/todos.ts:12 — `GET /todos` returns every row; no limit parameter
- Spec: default `limit=50`, max 200, validated; add `?cursor` passthrough but do NOT build
  full pagination (that's a product decision — see the 2026-07-06 PERF-03 note in the Journal)
- Acceptance:
  - [ ] `npx tsc --noEmit` exits 0
  - [ ] `curl -s localhost:3000/todos | jq length` returns ≤ 50 against the 60-row seed DB
  - [ ] `curl -s 'localhost:3000/todos?limit=9999' | jq length` returns ≤ 200 with HTTP 200 (clamped, not an error)

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
  - [ ] (provisional — gated on the wording decision; replace with a real criterion at unpark)

## Ledger
| ID | Status | Attempts | Commit | Note |
|---|---|---|---|---|
| SEC-01 | done | 2 | 3f9a2e1 | attempt 1 broke the seed fixture the new regression test relies on; see journal |
| SEC-02 | in-progress | 1 | — | resumed by session 2 |
| PERF-03 | todo | 0 | — | — |
| OPS-04 | needs-human | 0 | — | [OPS] requires staging server access |
| UX-05 | needs-human | 0 | — | [USER-DECISION] wording not decided |

## Journal
- 2026-07-06 run started (session 1). Baseline green: tsc 0, tests 34/34.
- 2026-07-06 SEC-01 attempt 1: scoping fix + the regression test the Spec names are in, but
  `seed.ts` assumed global visibility → 2 tests red. The fixture serves the new test, so
  fixing it stays inside the Spec (the diff is still one describable change). Re-ran.
- 2026-07-06 SEC-01 attempt 2: green (35/35 incl. new regression test). Committed 3f9a2e1
  (sha backfilled over `pending` in the next ledger write).
- 2026-07-06 PERF-03 note: full cursor pagination is deliberately out of scope — product
  decision pending; candidate future item if the team wants it.
- 2026-07-06 OPS-04: human-gated [OPS], parked as needs-human. UX-05: [USER-DECISION], parked.
- 2026-07-06 session 1 ended (context budget); ledger synced (`ratchet: sync ledger`).
  Resume: re-run /ratchet-loop on this file.
- 2026-07-07 run resumed (session 2). Ledger read: SEC-02 eligible (SEC-01 done). Marked
  in-progress. Suggestion for humans: consider a lint rule banning unscoped `findUnique` on
  owned models — recurring pattern behind SEC-01.
