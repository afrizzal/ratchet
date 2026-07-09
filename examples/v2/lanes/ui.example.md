# LANE ui — acme-todo API hardening
<!-- ratchet:v2:lane ui -->

> **Agent B's state file**, same moment as `api.example.md`. Agent B is a cheap model running
> unattended on `ratchet/ui`, in its own worktree, while agent A works `ratchet/api`. The two
> agents share a repo, a backlog, and a wave — and touch no common file.
>
> This lane is **OPEN** too (one `run started`, no `run ended` yet). Two lanes open at once is
> the normal state of a parallel wave, not an error.
>
> `UI-07` shows `Commit: pending` — the one place that value is legal. A commit cannot contain
> its own sha, so the atomic commit writes `pending` and the sha is backfilled by the *next*
> ledger write, here the run-ending `ratchet: sync ledger`. Legal because this lane is open;
> a **closed** lane file with `pending` in it is a defect.

## Ledger
| ID | Status | Attempts | Commit | Note |
|---|---|---|---|---|
| UI-07 | done | 1 | pending | sha backfills in the run-ending sync commit |

## Journal
- 2026-07-07 run started (lane ui, branch ratchet/ui)
- 2026-07-07 baseline green: tsc 0, tests 35/35 (worktree cut from master@2b1f9c3, which already
  carries the api lane's merged SEC-01).
- 2026-07-07 read every lane's journal at preflight, as a lane run must: api's session-1 entry
  warns that `seed.ts` assumes global visibility. No UI test touches it — but the trap is now
  *inherited*, not re-discovered. This is why reads are global while writes are lane-local.
- 2026-07-07 UI-07 attempt 1: green. Committed (sha pending until the next ledger write).
  Diff paths: `web/src/TodoList.tsx`, `web/src/TodoList.test.tsx` — both inside this lane's
  scope `web/src/**`, so nothing to park. A stray path in `src/routes/**` (the api lane) or in
  no lane at all (`package.json`) would have meant: revert the item, park it `needs-human`.
