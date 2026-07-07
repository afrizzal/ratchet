---
name: ratchet-backlog
description: Create, validate, extend, and groom ratchet BACKLOG.md files — from TODO/FIXME comments, GitHub issues, a PRD/notes file, or plain conversation. Enforces the ratchet:v1 contract, especially the core invariant "no acceptance criteria, no item". Use when the user says "make a backlog", "ratchet backlog", "validate the backlog", "add an item", or "turn these issues/TODOs into a backlog".
---

# Ratchet Backlog — the curator

The backlog file is the contract between whoever defines work and whoever (or whatever) executes it. Your job is to make that contract sharp: every item small enough for one commit, specific enough for a smaller model, and checkable enough that "done" is an observation, not an opinion.

Format authority: `docs/backlog-format.md` in the ratchet repo (github.com/afrizzal/ratchet); `examples/BACKLOG.example.md` there is a full worked example. Neither ships with an installed skill, so the compact contract is embedded here:

```markdown
# BACKLOG — <scope>
<!-- ratchet:v1 -->                      ← version marker, required

## Global checks                         ← optional; each line one backticked command
- `npx tsc --noEmit`

## Items                                 ← one ### block per item, one commit each

### SEC-01 — <title> [P0]                ← ID matches [A-Z]+-\d+, unique; priority [P0..P3]
- Tags: —                                ← only canonical gates: [RISKY] [OPS] [USER-DECISION] [BASELINE], or —
- Depends: —                             ← comma-separated IDs, all must be done first
- Evidence: path:line — <verified observation>
- Spec: <the exact minimal change — executors implement this and nothing more>
- Acceptance:                            ← ≥1; runnable commands or observable behaviors
  - [ ] `<command>` <expected result>

## Ledger                                ← one row per item; statuses: todo/in-progress/done/blocked/needs-human/skipped
| ID | Status | Attempts | Commit | Note |
|---|---|---|---|---|
| SEC-01 | todo | 0 | — | — |

## Journal                               ← append-only; - <date> <ID> <what happened>
```

Default file location: `ratchet/BACKLOG.md`.

## Modes

### `create from <source>`
Build a new backlog (or extend an existing one — never clobber Ledger history; new items get new IDs).

- **`todos`** — scan the codebase for `TODO|FIXME|HACK|XXX` markers. Each marker: read the surrounding code to reconstruct intent; markers too vague to spec go into a "needs definition" note for the user, not into items.
- **`github issues`** — `gh issue list --state open --json number,title,body,labels` (confirm `gh` works first). Map issue → item; keep `(#123)` in the title for traceability. Issues that are epics → split or park as needs-definition.
- **`<file>`** (PRD, meeting notes, audit report) — read it; extract discrete, committable units of work.
- **conversation** — interview the user briefly: what, where (files if known), how they'd verify it. Push back on vagueness once, then park it.

For every candidate item you MUST establish, in this order:
1. **Evidence/context** — a real `file:line` where the work anchors (open the file; verify). If no code anchor exists (new feature), anchor to the closest integration point.
2. **Spec** — the minimal change, one commit's worth. Bigger → split with `Depends:`.
3. **Acceptance** — ≥1 runnable command or observable behavior. If neither you nor the user can define one, the item does not enter the backlog; it enters a "## Needs definition" note at the bottom of your report.
4. **Tags** — `[USER-DECISION]` for product judgment, `[OPS]` for infra/credentials, `[RISKY]` for destructive/irreversible. When in doubt on destructive things, tag — the loop can always be re-run after a human unparks it.

Also detect the project's real verify commands (test/typecheck/lint from its manifest) and set `## Global checks` accordingly.

### `validate [path]`
Lint an existing file against the contract. Check, and report as a PASS/FAIL table with fixes:
- `ratchet:v1` marker present; all four sections present.
- IDs match `[A-Z]+-\d+`, unique; every item has Spec + ≥1 Acceptance criterion.
- **`Tags:` uses only canonical gates** (`[RISKY]`, `[OPS]`, `[USER-DECISION]`, `[BASELINE]`, or `—`). Any free-form tag (`auth`, `idor`, `perf`, bare `BASELINE` without brackets) is a **FAIL** — the loop won't recognize it, so a gated item reads as eligible.
- **At least one acceptance criterion is loop-runnable** — a backticked command the executor can actually run in this environment. An **ungated** item whose criteria are *all* manual/human-eye ("visit in a browser", "EXPLAIN shows Index Scan", "gh api returns …") or vague ("works well") is a FAIL: the loop can never mark it done and will bounce it to `needs-human`. Exception: an item carrying a human-gate tag may hold a *provisional* criterion — WARN, not FAIL — because executors never run gated items; it must be made real at unpark time.
- **Global checks are deterministic.** A check like `tsc --noEmit` that depends on a cache or generated types (passes warm, fails cold) is a FAIL — pin it (`--incremental false`, cold form). If the cold baseline is red, there must be a `[BASELINE]` item.
- Ledger: one row per item, statuses within the legal set (`todo/in-progress/done/blocked/needs-human/skipped`), every `done` row has a commit sha. `pending` is legal only on the most recent `done` row of a run still in flight — a file **at rest** with `pending` is a FAIL (fix: backfill the sha from `git log`).
- `Depends:` references exist and are acyclic.
- Smells (WARN, not FAIL): items that look multi-commit; **mixed-runnability items** — an ungated item where *some* criterion needs a human or infra the loop lacks (the loop parks on a *single* unverifiable criterion, so the item can never reach `done`; split it or make every criterion runnable); human-gate-worthy specs without tags (migrations, deletions, payments, auth changes); P0 items touching money/GL/auth/tenant that are *ungated* (they may be locally testable but a wrong fix corrupts data — flag for `[RISKY]` + `--verify fresh`); stale `in-progress` rows with no matching Journal tail (crash debris — the loop's preflight resets them to `todo`).
Fix mechanical problems only with the user's go-ahead; never rewrite Specs silently.

### `add "<description>"`
One item, conversationally: same four requirements as `create`. Append the item, its Ledger row, and nothing else.

### `groom`
With the user, pass over an existing backlog: merge duplicates, split oversized items, re-order by priority, retire stale items (status `skipped` with a note — never delete rows that have history), surface Journal suggestions the loop left behind ("the loop suggested a lint rule — want it as DX-07?"), and **unpark items the human has resolved** — per the format's Human transitions: journal the decision, clear the gate tag, make provisional criteria real, set status back to `todo` (reset `Attempts` for `blocked` items). Unparking is the one status edit that is yours to make, because the human is in the loop here.

## Rules

- **No acceptance criteria, no item.** This is the product's core invariant; you are its enforcement.
- Specs are written for the *least* capable executor that might run them — name files, name functions, state the expected diff shape.
- You edit Items and Ledger structure; you never fabricate Ledger *state* (attempts, commits) — that's the loop's.
- Journal is append-only even for you.

## Failure modes to avoid

- Laundering vague wishes into vague items (the #1 way loops fail downstream).
- Epics disguised as items.
- Deleting or renumbering items that have Ledger/Journal history.
- Inventing acceptance criteria the environment can't actually check.
