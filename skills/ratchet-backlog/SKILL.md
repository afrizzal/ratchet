---
name: ratchet-backlog
description: Create, validate, extend, groom, and migrate ratchet BACKLOG.md files — from TODO/FIXME comments, GitHub issues, a PRD/notes file, or plain conversation. Enforces the ratchet:v1 and ratchet:v2 contracts, especially the core invariant "no acceptance criteria, no item"; v2 adds parallel lanes (per-lane state files) for multi-agent execution. Use when the user says "make a backlog", "ratchet backlog", "validate the backlog", "add an item", "migrate the backlog to v2", or "turn these issues/TODOs into a backlog".
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

**Format v2 — parallel lanes** (marker exactly `<!-- ratchet:v2 -->`; full spec: `docs/backlog-format.md` §5 in the ratchet repo). The backlog holds `## Global checks` + `## Lanes` + `## Items` and **no Ledger/Journal**; each lane's state lives in `lanes/<lane>.md`, single writer, so parallel executors never write the same file:

```markdown
## Lanes                                 ← required in v2; first lane's Scope is the literal (rest) = the default lane
| Lane | Scope | Notes |
|---|---|---|
| core | (rest) | default lane — shared/cross-cutting work; barrier runs only |
| api | `src/api/**`, `prisma/**` | server hardening |

### SEC-01 — <title> [P0]                ← items gain one required field:
- Lane: api                              ← before Tags:; must name a declared lane
...
```

```markdown
# LANE api — <scope>                     ← lanes/api.md; created by YOU, never by the loop
<!-- ratchet:v2:lane api -->             ← marker must match filename + declaration

## Ledger                                ← exactly this lane's items' rows, in Items order
| ID | Status | Attempts | Commit | Note |
|---|---|---|---|---|
| SEC-01 | todo | 0 | — | — |

## Journal                               ← append-only; also carries run markers:
                                         ← - <date> run started (lane api, branch ratchet/api) / run ended (lane api)
```

Lane names match `[a-z][a-z0-9-]*`; named-lane scopes are disjoint backticked globs. An item's row and its lane file materialize together with the item — **you** create lane files; the loop refuses a declared lane whose file is missing.

**Human write windows (v2).** You (and the human you act for) write the backlog or a lane file only on the integration branch, and only while the affected lane is *closed* — its journal's last run marker is `run ended`, and no `ratchet/<lane>` branch sits unmerged ahead of the integration branch. While any lane run is open, the backlog is frozen (a Spec edited under a live run can merge cleanly next to a `done` row earned against the old Spec). If asked to edit anyway, refuse and name the open lane.

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

Emit v1 by default. Emit v2 when the user asks for lanes (or extends an existing v2 file): partition with the user (lanes follow the codebase's parallel-safe seams — disjoint path scopes, not team names), declare the default `(rest)` lane first, give every item a `Lane:`, and create each lane file with its rows. Items that add dependencies or touch paths no named lane owns go in the default lane. When extending an existing v2 backlog, new rows go into the item's lane file — respect the human write windows above.

### `validate [path]`
Lint an existing file against the contract. Check, and report as a PASS/FAIL table with fixes:
- Version marker present — exactly `ratchet:v1` (all four sections present) or exactly `ratchet:v2` (`## Lanes` + `## Items` present, **no** Ledger/Journal in the backlog). A marker containing `:lane` means this is a lane file, not a backlog — FAIL with a pointer to the real backlog.
- IDs match `[A-Z]+-\d+`, unique; every item has Spec + ≥1 Acceptance criterion.
- **v2 structural checks:** first lane's Scope is the literal `(rest)`, exactly one such lane; named-lane scopes disjoint (overlap = FAIL); lane names match `[a-z][a-z0-9-]*`; every item's `Lane:` names a declared lane; every declared lane has a `lanes/<lane>.md` whose `ratchet:v2:lane <name>` marker matches filename and declaration; every item has exactly one ledger row, residing in its own lane's file (missing, stranded, or duplicated rows = FAIL); run markers balance in a file at rest (an unbalanced `run started` with no run in flight = crash debris — WARN, the loop recovers it). Cross-scope checks apply only to **open** items (todo/in-progress/needs-human/blocked) in a **named** lane: Evidence/Spec paths in another named lane's scope = FAIL; paths in no named lane's scope, or a Spec naming a new dependency, = WARN "belongs in the default lane". Default-lane items are exempt — the `(rest)` barrier lane is where cross-scope work legally lives. `done`/`skipped` items are exempt too: history doesn't re-validate, which is what lets any v1 backlog migrate.
- **`Tags:` uses only canonical gates** (`[RISKY]`, `[OPS]`, `[USER-DECISION]`, `[BASELINE]`, or `—`). Any free-form tag (`auth`, `idor`, `perf`, bare `BASELINE` without brackets) is a **FAIL** — the loop won't recognize it, so a gated item reads as eligible.
- **At least one acceptance criterion is loop-runnable** — a backticked command the executor can actually run in this environment. An **ungated** item whose criteria are *all* manual/human-eye ("visit in a browser", "EXPLAIN shows Index Scan", "gh api returns …") or vague ("works well") is a FAIL: the loop can never mark it done and will bounce it to `needs-human`. Exception: an item carrying a human-gate tag may hold a *provisional* criterion — WARN, not FAIL — because executors never run gated items; it must be made real at unpark time.
- **Global checks are deterministic.** A check like `tsc --noEmit` that depends on a cache or generated types (passes warm, fails cold) is a FAIL — pin it (`--incremental false`, cold form). If the cold baseline is red, there must be a `[BASELINE]` item.
- Ledger: one row per item, statuses within the legal set (`todo/in-progress/done/blocked/needs-human/skipped`), every `done` row has a commit sha. `pending` is legal only on the most recent `done` row of a run still in flight — a file **at rest** with `pending` is a FAIL (fix: backfill the sha from `git log`). In v2, these rules apply to each lane file independently.
- `Depends:` references exist and are acyclic.
- Smells (WARN, not FAIL): items that look multi-commit; **mixed-runnability items** — an ungated item where *some* criterion needs a human or infra the loop lacks (the loop parks on a *single* unverifiable criterion, so the item can never reach `done`; split it or make every criterion runnable); human-gate-worthy specs without tags (migrations, deletions, payments, auth changes); P0 items touching money/GL/auth/tenant that are *ungated* (they may be locally testable but a wrong fix corrupts data — flag for `[RISKY]` + `--verify fresh`); stale `in-progress` rows with no matching Journal tail (crash debris — the loop's preflight resets them to `todo`).
Fix mechanical problems only with the user's go-ahead; never rewrite Specs silently.

### `add "<description>"`
One item, conversationally: same four requirements as `create`. Append the item, its Ledger row, and nothing else. On a v2 backlog: also assign `Lane:` (by scope match; dependency-adding or shared-path items → the default lane), write the row into that lane's file, and respect the human write windows — refuse while the lane is open.

### `groom`
With the user, pass over an existing backlog: merge duplicates, split oversized items, re-order by priority, retire stale items (status `skipped` with a note — never delete rows that have history), surface Journal suggestions the loop left behind ("the loop suggested a lint rule — want it as DX-07?"), and **unpark items the human has resolved** — per the format's Human transitions: journal the decision, clear the gate tag, make provisional criteria real, set status back to `todo` (reset `Attempts` for `blocked` items). Unparking is the one status edit that is yours to make, because the human is in the loop here. On a v2 backlog, every groom edit obeys the human write windows (integration branch only, affected lane closed); re-assigning an item's `Lane:` means moving its row to the new lane's file in the same edit — a row stranded in the wrong lane file makes the loop refuse the whole backlog.

### `migrate [path]`
v1 → v2, human-invoked. Refuse while a run is in flight (`in-progress` rows — have the loop's preflight recover them first). An at-rest `pending` sha is not a blocker: backfill it from `git log`, journaling the fix. Then, in one commit (`ratchet: migrate backlog to v2`):
1. Partition with the user; write `## Lanes` (default `(rest)` lane first) and add `Lane:` to every item. Open items whose paths cross the new scopes, or whose Spec adds dependencies, go to the default lane.
2. Move each Ledger row verbatim into its item's lane file.
3. **Copy** each Journal line into the lane file of every item ID it mentions (whole-token match `[A-Z]+-\d+` — never substring; duplication across lanes is harmless in an append-only history). Lines mentioning no item ID go to the default lane.
4. Remove Ledger/Journal from the backlog; flip the marker to `<!-- ratchet:v2 -->`; run `validate` on the result and show the report.
Migration never renumbers, rewrites, or drops a line — items keep their IDs and history.

## Rules

- **No acceptance criteria, no item.** This is the product's core invariant; you are its enforcement — identical in every lane; v2 changes where state lives, never what done means.
- Specs are written for the *least* capable executor that might run them — name files, name functions, state the expected diff shape.
- You edit Items and Ledger structure; you never fabricate Ledger *state* (attempts, commits) — that's the loop's. In v2 you are also the only writer of lane *files* (creation, row insertion, unpark edits), always inside the human write windows.
- Journal is append-only even for you.

## Failure modes to avoid

- Laundering vague wishes into vague items (the #1 way loops fail downstream).
- Epics disguised as items.
- Deleting or renumbering items that have Ledger/Journal history.
- Inventing acceptance criteria the environment can't actually check.
