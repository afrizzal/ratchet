# LANE api — acme-todo API hardening
<!-- ratchet:v2:lane api -->

> **Agent A's state file.** Snapshot taken while two agents run at once (see `../README.md`):
> agent A is here on `ratchet/api`, agent B is in `../ui.example.md` on `ratchet/ui`. Neither
> can write the other's file, so neither can conflict with the other.
>
> This lane is **OPEN** — scanning the journal bottom-up, the first run marker is `run started`.
> That is the liveness signal: agent B, a crashed session, and `/ratchet-recommend` all read it
> and know this lane belongs to somebody right now. Note there are two `run started` and one
> `run ended`: session 1 opened and closed cleanly; session 2 is still going.

## Ledger
| ID | Status | Attempts | Commit | Note |
|---|---|---|---|---|
| SEC-01 | done | 2 | 3f9a2e1 | attempt 1 broke the seed fixture the new regression test relies on; see journal |
| SEC-02 | in-progress | 1 | — | agent A, session 2 |
| PERF-03 | todo | 0 | — | — |
| UX-05 | needs-human | 0 | — | [USER-DECISION] wording not decided |

## Journal
- 2026-07-06 run started (lane api, branch ratchet/api)
- 2026-07-06 baseline green: tsc 0, tests 34/34.
- 2026-07-06 SEC-01 attempt 1: scoping fix + the regression test the Spec names are in, but
  `seed.ts` assumed global visibility → 2 tests red. The fixture serves the new test, so
  fixing it stays inside the Spec (the diff is still one describable change). Re-ran.
- 2026-07-06 SEC-01 attempt 2: green (35/35 incl. new regression test). Committed 3f9a2e1
  (sha backfilled over `pending` in the next ledger write).
- 2026-07-06 PERF-03 note: full cursor pagination is deliberately out of scope — product
  decision pending; candidate future item if the team wants it.
- 2026-07-06 UX-05: human-gated [USER-DECISION], parked as needs-human.
- 2026-07-06 run ended (lane api). Session 1 closed; merged to master as 2b1f9c3.
- 2026-07-07 run started (lane api, branch ratchet/api)
- 2026-07-07 baseline green. SEC-02 eligible: SEC-01 is `done` **in this lane** — a same-lane
  dependency needs no merge. (A dependency in another lane would have to be done *and merged*
  before it counts.) Marked in-progress, attempt 1.
- 2026-07-07 SEC-02 suggestion for humans: consider a lint rule banning unscoped `findUnique`
  on owned models — the recurring pattern behind SEC-01. (Executors suggest; they never add
  items themselves.)
