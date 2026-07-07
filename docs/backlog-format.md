# The Ratchet Backlog Format (v1)

The backlog file is Ratchet's public contract. Skills come and go; this format is what makes loops verified, resumable, and portable across agents. Keep it boring and keep it stable.

A conforming file is plain markdown with four parts:

```markdown
# BACKLOG — <project or scope name>
<!-- ratchet:v1 -->

## Global checks          ← optional
## Items                  ← required
## Ledger                 ← required
## Journal                ← required (may start empty)
```

The HTML comment `<!-- ratchet:v1 -->` near the top is the version marker. Executors must refuse files without it rather than guess.

---

## 1. Global checks (optional)

Commands that must pass **after every item**, in addition to that item's own acceptance criteria. This is the regression gate.

```markdown
## Global checks
- `npx tsc --noEmit`
- `npm test`
```

Rules:
- Each line is one runnable command in backticks.
- The executor establishes a **green baseline** by running these once before the first item. If the baseline is red, the loop must stop — you cannot ratchet forward from a broken start (exception: items tagged `[BASELINE]`, which exist to fix the baseline itself).

## 2. Items

One `###` block per unit of work. **An item must be completable in a single atomic commit** — if it can't, it's two items.

```markdown
### SEC-01 — Scope item lookups by tenant [P0]
- Tags: —
- Depends: —
- Evidence: src/api/items/[id]/route.ts:16 — GET/PATCH/DELETE filter by group only, not tenant
- Spec: add tenantId (from the session helper, never from the request body) to all three
  where-clauses; use updateMany/deleteMany so the write is scoped too
- Acceptance:
  - [ ] `npx tsc --noEmit` exits 0
  - [ ] request against another tenant's item id returns 404 (manual or scripted check)
  - [ ] the existing save/delete flow in the UI still works
```

Field contract:

| Field | Required | Rules |
|---|---|---|
| Heading | yes | `### <ID> — <Title> [P0..P3]`. ID matches `[A-Z]+-\d+`, unique in the file. Priority in brackets. |
| `Tags:` | yes (may be `—`) | Human gates: `[USER-DECISION]` (product/design call), `[OPS]` (needs infra access/credentials), `[RISKY]` (irreversible or wide blast radius). Special: `[BASELINE]` (exempt from the green-baseline rule because it fixes the baseline). Executors **never** work on human-gated items. |
| `Depends:` | no | Comma-separated item IDs. An item is eligible only when all dependencies are `done`. |
| `Evidence:` | yes | `path:line — what was observed`. Line numbers are anchors from authoring time; executors must re-verify before editing. If evidence no longer holds, the item goes to `needs-human`, not to improvisation. |
| `Spec:` | yes | The exact, minimal change. The executor implements the Spec and *nothing more* — no drive-by refactors, no new dependencies unless the Spec names them. |
| `Acceptance:` | yes, ≥1 | Checkbox list. Each criterion is either a backticked command with its expected result, or an observable behavior a session can actually check. **No acceptance criteria, no item.** Vague criteria ("works well") are a **validation error on any ungated item**. Exception: an item carrying a human-gate tag (`[USER-DECISION]`/`[OPS]`/`[RISKY]`) may hold a *provisional* criterion (e.g. "(to be defined with the wording decision)") — executors never run gated items, so validators report this as a warning, not an error. The criterion must be made real when the item is unparked. |

Items are **immutable to executors**. Humans edit items; the loop only writes to the Ledger and Journal. In particular: an executor may never edit acceptance criteria — if a criterion is wrong or unverifiable, the item becomes `needs-human` with a journal note. The goalposts don't move.

## 3. Ledger

The single source of truth for state. One row per item, same order as Items.

```markdown
## Ledger
| ID | Status | Attempts | Commit | Note |
|---|---|---|---|---|
| SEC-01 | done | 2 | a1b2c3d | attempt 1 failed the tenant check; attempt 2 green — see journal |
| PERF-02 | todo | 0 | — | — |
| OPS-03 | needs-human | 0 | — | [OPS] requires prod .env access |
```

Status machine (executors may only follow these arrows):

```
todo → in-progress → done
                   → blocked        (attempts exhausted; item's changes reverted)
                   → needs-human    (stale evidence, wrong criteria, spec wrong, too large)
todo → needs-human                  (executor: human-gate tag present, or high-stakes gate unsatisfied)
todo → skipped                      (human decision only — executors never skip)

in-progress → todo                  (recovery only: a resuming executor, at preflight,
                                     after confirming no uncommitted source changes remain)
needs-human → todo                  (human only — see Human transitions below)
blocked     → todo                  (human only — see Human transitions below)
```

Rules:
- Mark `in-progress` **before** starting work, and set `Attempts` to 1 with it (a crash then leaves a visible trace).
- **`Attempts` counts implement+verify cycles.** The first implementation pass is attempt 1; each further fix cycle increments it. The cap (default 3) is on this total — a red verify at attempt 3 means `blocked`.
- `done` requires: item acceptance green + global checks green + one atomic commit that contains both the change **and** this ledger row flipped to `done`.
- **The sha backfill.** A commit's own sha cannot appear inside that commit. So the atomic commit records `pending` in the `Commit` column; immediately after committing, the executor writes the short sha over `pending` as an uncommitted edit that rides along in its *next* ledger write — the next item's commit, a block-recording commit, or the run-ending sync commit. Every run ends by committing any outstanding ledger edits as `ratchet: sync ledger`, so a file **at rest** never contains `pending`. Validators: `pending` is legal only on the most recent `done` row of a run still in flight; anywhere else it is a defect (fix: backfill the sha from `git log`).
- `blocked` requires: the item's working-tree changes fully reverted, and a Journal entry saying what failed and the current hypothesis.

### Human transitions (unparking)

Humans release parked items; executors never do. To unpark an item:

1. Resolve the reason it parked (answer the decision, provide the credential, review and approve the risk) and record it as a Journal line — `- <date> <ID> unparked: <decision>`.
2. If the item is tag-gated, edit `Tags:` — remove the gate (or replace it with `—`). A gate tag left in place re-parks the item on the next run regardless of status.
3. Make the acceptance criteria loop-runnable: replace provisional placeholders with real criteria, and rewrite (or check yourself, noting it in the Journal) any criterion only a human or infrastructure can verify — the loop parks an item over a *single* criterion it cannot check.
4. Set the Ledger status to `todo`. For `blocked` items, also reset `Attempts` to 0 (or add a Note raising the cap).

The next loop run then picks the item up through normal eligibility.

## 4. Journal

Append-only. This is what makes resume cheap — a fresh session reads it and inherits every lesson without re-discovering them.

```markdown
## Journal
- 2026-07-07 SEC-01 attempt 1: tsc green but tenant check failed — list route shares the
  same unscoped query; widened spec interpretation? NO — stayed in spec, fixed only [id]
  route, list route filed as SEC-02 suggestion for the human.
- 2026-07-07 SEC-01 attempt 2: green. Committed a1b2c3d.
```

Rules:
- One `- <date> <ID> <what happened / what was learned>` line per meaningful event (attempt result, block reason, discovery worth keeping).
- Never rewrite or delete existing lines.
- Discoveries that suggest *new* items go in the Journal as suggestions — executors do not add items themselves.

## Conformance summary for executors

1. Refuse files without the `ratchet:v1` marker or with **structural** damage (missing sections, duplicate IDs, an item with no Spec or no Acceptance). Per-item defects that don't break structure — stale evidence, unrunnable criteria on an ungated item — park **that item** as `needs-human`, not the file.
2. Eligible item = status `todo` + no human-gate tag + all `Depends:` done. Take them in file order (priority is expressed by ordering and the `[P*]` label humans assigned).
3. Edit only Ledger + Journal. Items are read-only. The one backward move executors may make: `in-progress → todo` at preflight, to recover an item orphaned by a crashed run.
4. One item = one commit = code change + ledger update together (`Commit: pending`, sha backfilled in the next ledger write — see §3).
5. Stop conditions: no eligible items; two consecutive `blocked`; an explicit item/time budget from the invoker; or context exhaustion (write state, report how to resume).
6. End every run with the ledger fully committed — `ratchet: sync ledger` if needed. No `pending` shas and no dirty tree survive a finished run.

Anything not specified here is executor freedom — but when in doubt, prefer the boring choice: it keeps ports compatible.
