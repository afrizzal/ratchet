# The Ratchet Backlog Format

The backlog file is Ratchet's public contract. Skills come and go; this format is what makes loops verified, resumable, and portable across agents. Keep it boring and keep it stable.

Two versions exist. **v1** (one file, one executor at a time) is sections 1–4 plus the conformance summary — it is not deprecated and remains the default. **v2** (section 5) relocates mutable state into per-lane files so several executors can work the same backlog concurrently; everything v1 says about items, statuses, attempts, and the sha backfill holds in v2 unchanged, per lane file.

A conforming **v1** file is plain markdown with four parts:

```markdown
# BACKLOG — <project or scope name>
<!-- ratchet:v1 -->

## Global checks          ← optional
## Items                  ← required
## Ledger                 ← required
## Journal                ← required (may start empty)
```

The HTML comment near the top is the version marker: exactly `<!-- ratchet:v1 -->` or exactly `<!-- ratchet:v2 -->`. Executors must refuse files without a marker they support rather than guess — and a marker containing `:lane` identifies a v2 *lane file* (§5.2), never a backlog.

---

## 1. Global checks (optional)

Commands that must pass **after every item**, in addition to that item's own acceptance criteria. This is the regression gate.

```markdown
## Global checks
- `npx tsc --noEmit --incremental false`
- `npm test`
```

Rules:
- Each line is one runnable command in backticks, pinned to a **deterministic** form — a check that passes warm and fails cold (an incremental cache, dev-server-generated types) is a false gate.
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

1. Refuse files without a version marker you support (`ratchet:v1`, or `ratchet:v2` if you implement §5) or with **structural** damage (missing sections, duplicate IDs, an item with no Spec or no Acceptance). Per-item defects that don't break structure — stale evidence, unrunnable criteria on an ungated item — park **that item** as `needs-human`, not the file.
2. Eligible item = status `todo` + no human-gate tag + all `Depends:` done. Take them in file order (priority is expressed by ordering and the `[P*]` label humans assigned).
3. Edit only Ledger + Journal. Items are read-only. The one backward move executors may make: `in-progress → todo` at preflight, to recover an item orphaned by a crashed run.
4. One item = one commit = code change + ledger update together (`Commit: pending`, sha backfilled in the next ledger write — see §3).
5. Stop conditions: no eligible items; two consecutive `blocked`; an explicit item/time budget from the invoker; or context exhaustion (write state, report how to resume).
6. End every run with the ledger fully committed — `ratchet: sync ledger` if needed. No `pending` shas and no dirty tree survive a finished run.

Anything not specified here is executor freedom — but when in doubt, prefer the boring choice: it keeps ports compatible.

---

## 5. Format v2 — parallel lanes (`ratchet:v2`)

v2 exists for one reason: **several executors working the same backlog at the same time without ever writing the same file.** It changes *where state lives* and adds the machinery that makes parallel writes safe. It never changes what *done* means: every item in every lane still carries ≥1 acceptance criterion (**no acceptance criteria, no item** — same field contract, same validation, same gated-item provisional exception as §2), and the status machine, attempts accounting, sha backfill, high-stakes gate, and human-only unparking are exactly v1's, applied per lane file.

v1 files stay valid forever. v2 is opt-in — usually via `/ratchet-backlog migrate` (§5.8).

### 5.1 Layout and ownership

```
ratchet/
  BACKLOG.md          ← human-owned; executors NEVER write it
  lanes/
    core.md           ← the default lane's state file (ledger + journal)
    api.md            ← one state file per lane; single writer
    ui.md
  NEXT.md             ← recommend's routed plan; integration-branch only (§5.6)
```

A v2 `BACKLOG.md` (marker exactly `<!-- ratchet:v2 -->`) has three sections — `## Global checks` (optional, shared by all lanes), `## Lanes` (required), `## Items` (required) — and **no Ledger or Journal sections**. State lives in one file per lane under `lanes/`.

One term, used throughout: the **integration branch** is the branch lane branches merge back into, and the only place `BACKLOG.md` and `NEXT.md` are written. It is the branch the backlog was last committed on — the repo's default branch, unless the human is running the whole effort on a work branch (`ratchet/<scope>`), in which case that is it. When an executor cannot determine it, it says so and asks rather than guessing; lane branches are always named `ratchet/<lane>`, so anything else that the backlog's history sits on is the integration branch.

Ownership is the whole point:

| File | Written by | Notes |
|---|---|---|
| `BACKLOG.md` | humans (and the curator/audit skills acting for them) | frozen while any lane run is open (§5.5) |
| `lanes/<lane>.md` | exactly one writer at a time | an executor during its run; a human only per §5.5 |
| `NEXT.md` | recommend, on the integration branch only | never written or committed by a lane run |

### 5.2 Lanes and lane files

```markdown
## Lanes
| Lane | Scope | Notes |
|---|---|---|
| core | (rest) | default lane — shared/cross-cutting work; barrier runs only |
| api | `src/api/**`, `prisma/**` | server hardening |
| ui | `src/components/**`, `src/app/**` | frontend debt |
```

- Lane names match `[a-z][a-z0-9-]*` (they become filenames and branch names).
- The **first-declared lane is the default lane** and its Scope is the literal `(rest)`: every path not claimed by another lane. Exactly one `(rest)` lane, always first. A v2 backlog with only the default lane is legal (single-lane v2).
- Named lanes declare comma-separated backticked path globs. **Scopes must be disjoint** — overlapping globs are a validation FAIL.
- Every item carries a new required field `- Lane: <name>` (placed before `Tags:`), naming a declared lane.

Each declared lane has a state file `lanes/<lane>.md`:

```markdown
# LANE api — <project or scope name>
<!-- ratchet:v2:lane api -->

## Ledger
| ID | Status | Attempts | Commit | Note |
|---|---|---|---|---|
| SEC-01 | todo | 0 | — | — |

## Journal
- <date> <ID or run event> <what happened>
```

The marker embeds the lane name and must match both the filename and the `## Lanes` declaration. The Ledger holds exactly the rows for the items whose `Lane:` names this lane, in Items order. All §3 rules (status machine, attempts, `Commit: pending` + sha backfill in this file's next ledger write, no `pending` at rest) and §4 rules (append-only journal) apply to each lane file independently.

**Lane files are created only by the curator** (`/ratchet-backlog create`, `add`, `migrate`) — an item's row and its lane file materialize together with the item. The loop never creates lane files; a declared lane whose file is missing is structural damage (§5.9).

### 5.3 Run markers — the liveness signal

Concurrency needs a mechanical answer to "is this lane live right now?". Two journal line shapes provide it:

```markdown
- 2026-07-09 run started (lane api, branch ratchet/api)
- 2026-07-09 run ended (lane api)
```

A lane is **open** when, scanning its journal bottom-up, the first run marker found is `run started`. Rules:

- A run appends and **commits** `run started` at preflight, before any item work (`ratchet(<lane>): run open`).
- Every run ends by appending `run ended`, riding in the final `ratchet: sync ledger` commit. A crashed run leaves the lane open — visibly.
- A new run finding its own lane open: if the marker names **the current branch**, that is crash debris — journal `found open run from <date> — presumed crashed, recovering`, run v1 preflight recovery, and continue **under the existing marker**. It appends no second `run started`; its own `run ended` balances the one already there. If the marker names a **different branch**, the lane may be live elsewhere — STOP and report; a human decides.
- Markers are per lane. A sequential (no-lane) run opens each lane as it begins working it and closes it in the `ratchet: sync ledger (<lane>)` commit it makes before switching to the next.
- A run that **refuses to start** (barrier preconditions unmet, lane open elsewhere, structural damage) writes no marker at all: liveness is decided before the first write, so a refused run never leaves a lane open. Conversely, once a run has opened its lane, **every** exit closes it — a red baseline, a blocked item, context exhaustion — because a deliberate stop that leaves the lane open is indistinguishable from a crash.
- No executor ever opens, closes, or "recovers" a lane other than its own.
- At rest, every lane file's markers balance: equal `run started` and `run ended`, `run ended` last. An unbalanced tail with no run in flight is crash debris — a warning, and the next run on that lane recovers it.

### 5.4 The concurrency model

- **Single writer per lane.** A run's only write targets are its own lane file and the source paths of its items. It **reads** everything: `BACKLOG.md`, and all lane files — other ledgers to evaluate `Depends:`, other journals for inheritance (a lane run's preflight reads *all* journals; lessons don't respect scope boundaries). Run-level events (baseline results, recovery notes) are journaled in the run's own lane file.
- **One worktree + branch per concurrent lane.** Convention: branch `ratchet/<lane>`, one git worktree per lane. Two runs must never share a worktree.
- **The lane branch is the authority.** Until merged, the authoritative `lanes/<lane>.md` is the tip of `ratchet/<lane>`. A `--lane X` run resumes the existing `ratchet/X` branch (and its worktree, if present) when the branch exists; only when it doesn't does the run create it from the integration branch head. Never start a lane from the integration branch while an unmerged `ratchet/X` exists — that forks the lane's state.
- **Freshness before work.** At preflight, if the lane branch is missing commits from the integration branch, merge the integration branch in first (this is how merged cross-lane `Depends:` and `[BASELINE]` fixes become visible). If that merge conflicts, STOP for a human.
- **Cross-lane `Depends:`** are satisfied only by a `done` row *visible in the current worktree* — done-and-merged. An unmerged done on another lane's branch does not count. This is conservative on purpose; recommend sequences cross-lane-dependent items into post-merge waves.
- **Scope binds the diff, not just the Spec.** Validation reads declared paths; conflicts come from produced diffs. So before committing an item, a named-lane executor lists the diff's paths: every source path must lie inside **its own** lane's scope. A path in another lane's scope — or in *no* lane's scope, which means it belongs to the default lane (new dependencies and their lockfile, shared config, generated schema) — means revert the item and park it `needs-human` ("diff escapes lane scope — refile in the default lane"). A Spec that names a new dependency therefore belongs in the default lane from the start, and validation says so (§5.9). Two lanes each adding a dependency in one wave is precisely the lockfile conflict this check prevents.
- **The default lane runs as a barrier.** Because its scope is `(rest)` — and its items (integration fixes, dependency bumps, repo-wide changes) may legitimately touch anything — a default-lane run requires: every other lane **closed** (no open run markers) and **merged** (no `ratchet/<lane>` branch ahead of the integration branch). This is checked *before* the run opens its own lane, so a barrier run that may not proceed leaves nothing behind. Under a barrier, the scope check is waived for the default lane. The same preconditions apply to a sequential no-`--lane` run over a v2 file, which works lanes one at a time in a single worktree (committing `ratchet: sync ledger (<lane>)` before switching lane files, so at most one lane file is ever dirty).
- **[BASELINE] across lanes.** A red baseline blocks every lane. The `[BASELINE]` item's own lane runs it first (mandatory first pick, as v1); a run on any *other* lane must STOP and report "baseline-blocked by `<ID>` in lane `<Y>` — run that lane first, merge, then relaunch". Waiting is a stop, not a write.

What this buys: lane files have one writer each, source scopes are disjoint, and everything shared is serialized through the default lane's barrier — so merging lane branches is conflict-free *for everything the format controls*. What it does not buy: semantic coupling. Two lanes can each be green in isolation and red together; that is what integration (§5.7) is for.

### 5.5 Human write windows

Humans still own the work definition, and some human edits must touch lane files (unparking flips a status row; adding an item adds its row; grooming resets attempts). The rule that keeps these from racing a live run:

> **Humans (and curator skills) write `BACKLOG.md` or any lane file only on the integration branch, and only while the affected lane is closed** — no open run marker, no unmerged `ratchet/<lane>` branch. While any lane run is open, `BACKLOG.md` is frozen: a Spec edited under a live run can otherwise merge cleanly next to a `done` row that was earned against the old Spec, and nothing would flag it.

The v1 Human transitions procedure (§3) is otherwise unchanged in v2 — same steps, with the status/attempts/journal edits landing in the item's lane file, under this window rule.

### 5.6 NEXT.md in v2

`NEXT.md` is regenerated wholesale by recommend, which makes it a guaranteed merge conflict if lane branches carry their own copies. So in v2 it is **single-home**: recommend runs on the integration branch only (it warns and refuses routing when unmerged lane branches make its read stale), and lane runs never write, commit, or regenerate `NEXT.md`. Wave invocations for a v2 backlog always carry both `--lane <name>` and an explicit `--only <IDs>` — `--lane` selects the state file and worktree, it is *not* routing and never satisfies the high-stakes gate's `--only` requirement.

### 5.7 Integrating lanes

Merging is a **human step** (or `/ratchet-ship` acting under human direction); no executor merges branches.

1. Confirm the lane is closed (its journal's last run marker is `run ended`).
2. Merge `ratchet/<lane>` into the integration branch — **merge commit or fast-forward only**. Never squash, never rebase-merge: both rewrite shas and orphan every `Commit:` cell in the merged ledger.
3. After each merge (or batch of merges), run the Global checks once on the integration branch. Lanes green in isolation can still be red together.
4. A red integration is **new work, not a rollback**: journal the failure in the default lane's file, and file the fix as a new item — normally `Lane: <default>`, since interaction fixes tend to cross scopes and the default lane's barrier serializes them safely. Filing items is a human/curator act; an executor at a merge point only reports.

### 5.8 Migration (v1 → v2)

Human-invoked, via `/ratchet-backlog migrate`. Refuse when a run is in flight (`in-progress` rows). An at-rest `pending` sha is not a blocker — backfill it from `git log` first, journaling the fix. Then:

1. The human chooses the partition; the curator writes `## Lanes` (default lane first) and adds `Lane:` to every item.
2. Each Ledger row moves verbatim to its item's lane file.
3. Journal lines are **copied** into the lane file of every item ID they mention (whole-token match `[A-Z]+-\d+`; duplication across lanes is harmless in an append-only history). Lines mentioning no item ID go to the default lane.
4. Ledger/Journal sections are removed from BACKLOG.md; the marker flips to `<!-- ratchet:v2 -->`. One commit: `ratchet: migrate backlog to v2`.

Items keep their IDs and history — migration never renumbers, rewrites, or drops a line. A `done` item whose old Evidence/Spec paths now cross lane scopes is history, not a defect (§5.9 scope checks apply to open items only).

### 5.9 Conformance additions for v2 executors and validators

Everything in the v1 conformance summary, plus:

1. **Structural** (refuse the run, point at `/ratchet-backlog validate`): missing/duplicate `(rest)` lane; overlapping named-lane scopes; an item with no `Lane:` or an undeclared lane name; a declared lane with no lane file, or a lane-file marker that doesn't match its filename/declaration; an item with no ledger row anywhere, a row in the wrong lane's file, or duplicate rows across files. These are cross-file integrity breaks — there is no per-item park target for them.
2. Scope-of-refusal for a `--lane X` run: defects in `BACKLOG.md`, in `lanes/X.md`, or in the row-residency of *X's own items* are STOPs. Internal damage confined to another lane's file is a loud WARN — report it, never touch it.
3. Per-item validation is v1's, unchanged — Spec required, **≥1 acceptance criterion required**, canonical tags only — identical in every lane.
4. Cross-scope checks on **open** items (todo/in-progress/needs-human/blocked), and only on items in a **named** lane: an item whose Evidence/Spec paths fall in another named lane's scope is a FAIL; paths in no named lane's scope, or a Spec naming a new dependency, WARN with "belongs in the default lane". **Default-lane items are exempt** — the `(rest)` lane is the barrier where cross-scope work legally lives, so a default-lane item may cite any path (that is what makes an integration fix, or a dependency bump touching one component, representable at all). `done`/`skipped` items are exempt too: history doesn't re-validate, so migrating a v1 backlog whose old items span the new partition is always possible.
5. `pending`-sha and dirty-tree rules apply per lane file; run markers must balance in any file at rest.
6. Invocation sanity: `--lane` on a v1 file is an error; `--only` IDs that exist but belong to a different lane than `--lane` names are an **error** ("SEC-01 is in lane ui — re-run with --lane ui or regenerate NEXT.md"), never a silent "backlog dry".

A full worked v2 example ships at `examples/v2/`, captured mid-flight with **two agents running concurrently**: the backlog, the routed plan that produced the parallel wave, two open lane files (one holding the only legal `pending` sha), and the untouched barrier lane. Its `README.md` walks the whole timeline.
