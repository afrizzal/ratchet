---
name: ratchet-loop
description: Execute a ratchet BACKLOG.md in a verified engineering loop — pick the next eligible item, implement exactly its spec, run its acceptance criteria until green (bounded attempts), lock it in as one atomic commit, update the ledger, repeat until the backlog is dry or a stop condition fires. Use when the user says "run the backlog", "ratchet", "work through BACKLOG.md", or wants autonomous-but-verified execution of a prepared item list. Requires a backlog conforming to the ratchet:v1 format.
---

# Ratchet Loop — the executor

You are executing a ratchet backlog. Your contract is **forward motion only**: an item is either done-verified-committed, or it is not done and the Ledger says exactly why. Nothing in between survives your run.

The backlog file is the single source of state. You are stateless; treat every run as if a different agent did the previous one (often true). Everything you learn that matters goes in the file, not in your head.

**Load the executor rules first.** Read `executor-rules.md` from this skill's own directory (the folder this SKILL.md lives in) before touching any item — 12 binding, do-this-exactly rules distilled from field runs. When a rule and your instinct disagree, follow the rule.

## Invocation

```
/ratchet-loop [path] [--max-items N] [--only ID[,ID...]] [--verify inline|fresh] [--dry-run]
```

- `path` — backlog file. Default: `ratchet/BACKLOG.md`, else `BACKLOG.md` at repo root. If neither exists, say so and suggest `/ratchet-audit` or `/ratchet-backlog create`. The resolved path is **the backlog file** everywhere below, and its containing directory is the loop's **state directory** (default `ratchet/`).
- `--max-items N` — stop after N items reach a terminal state this run.
- `--only` — restrict eligibility to the listed IDs (dependencies still respected). Also the explicit-routing signal the high-stakes gate requires — see below.
- `--verify` — verification mode, see "Verification modes" below. Default `inline`.
- `--dry-run` — do Phase 0 only: validate, report the execution plan and any recovery a real run would perform, **write and commit nothing**.

## Phase 0 — Preflight (never skip, never reorder)

1. **Read the backlog file in full and validate its structure:** the `<!-- ratchet:v1 -->` marker near the top; all four sections (`## Items`, `## Ledger`, `## Journal`, optionally `## Global checks`); IDs matching `[A-Z]+-\d+` and unique; every item has a Spec and ≥1 Acceptance criterion; a Ledger row per item with a status from the legal set (`todo` / `in-progress` / `done` / `blocked` / `needs-human` / `skipped`). Structurally broken → STOP, report the exact problems, suggest `/ratchet-backlog validate`. Never guess at a malformed file. Per-item defects that don't break structure — stale evidence, unrunnable criteria on an *ungated* item — are handled later by parking **that item**, not by refusing the file. (Full contract: `docs/backlog-format.md` in the ratchet repo; the checks you need are the ones listed here.)
2. **Require a clean working tree — with the loop-state exception.** `git status` must show no staged or unstaged changes to tracked files. Classify any dirt you find:
   - The backlog file and the state directory are the loop's *own* state. **Untracked** → not user dirt; they will be committed in step 3. **Dirty but tracked** → state left by a previous run (a crash, or an unsynced sha backfill); journal one line and let step 3 commit it.
   - Dirty *source* paths that belong to an `in-progress` Ledger item are crash debris, handled in step 3 — attribute them via the Journal tail and the item's Evidence/Spec; if the attribution is ambiguous, STOP and ask.
   - Any other dirty tracked file → STOP and ask the user (their work is not yours to stash). This invariant is what makes your failure-recovery safe.
3. **Recover, then commit the recovery once.** A Ledger row stuck at `in-progress` is debris from a crashed run. For each one: revert exactly the source paths attributed to it in step 2 (executor rule 9) if any remain; journal `found in-progress from a crashed run — reset to todo`; set the row back to `todo`. This is the only backward status move an executor may make, and only at preflight. Then make **one** commit covering everything steps 2–3 touched — first-run state (`ratchet: add backlog`) or recovery (`ratchet: recover state`). If steps 2–3 found nothing, there is nothing to commit.
4. **Read the Ledger and the entire Journal.** The Journal is your inheritance — previous attempts, known traps, failed hypotheses. Do not re-attempt something the Journal says failed without a new reason.
5. **Establish the green baseline.** Run every command under `## Global checks` once, **in its deterministic form** — cold, no incremental cache, no reliance on a running dev server's generated types (a warm `tsc --noEmit` can report green while the same check run cold, as CI runs it, is red). If a Global check looks cache-sensitive and the backlog didn't pin it, run the cold form yourself and note the discrepancy in the Journal. Red baseline → STOP and report (you cannot ratchet forward from a broken start) — unless a `[BASELINE]`-tagged item exists, in which case that item is your mandatory first pick, ahead of file order.
6. **Detect the repo's commit convention** from `git log --oneline -15` (conventional commits? plain sentences? scope prefixes?). You will match it.
7. **Check the branch.** If you are on the repo's default branch and the repo is PR-based or push-to-deploy, propose creating a work branch (`ratchet/<scope-or-date>`) before the first commit — `/ratchet-ship` opens the PR later. Proceed on the default branch only with the user's explicit OK.
8. **Read `NEXT.md` (in the state directory) if it exists.** It is the routed plan from `/ratchet-recommend`: honor its per-item verify assignments — an item it routes SUPERVISED or SENIOR requires `--verify fresh`. If fresh is not in effect, park **only** the SUPERVISED/SENIOR items that would otherwise execute *in this run* (they pass eligibility and match `--only`); items outside this run's scope are left untouched — their wave commands carry the right flags already. If NEXT.md predates the last Ledger change, say it is stale and suggest regenerating it. Explicit flags from the invoker override NEXT.md.
9. **Announce the plan:** eligible items in order, parked items with reasons, the branch, the stop conditions in effect. In `--dry-run`, stop here — and since a dry run writes nothing, *report* the commits steps 2–3 would have made instead of making them.

## Eligibility

An item is eligible when ALL hold: Ledger status `todo` · no human-gate tag (`[USER-DECISION]`, `[OPS]`, `[RISKY]`) · every `Depends:` ID is `done` · matches `--only` if given. Take eligible items in file order. Human-gated `todo` items: set their Ledger status to `needs-human` with the tag as the note (once), and move on. Parked items stay parked until a **human** unparks them — status back to `todo`, gate tag removed, decision journaled (the format's "Human transitions" procedure); you never do this yourself.

**High-stakes gate (mechanical).** An item is *high-stakes* when its Spec or Evidence touches any of the trigger list (verbatim from executor rule 1): **auth/permissions, tenant boundaries (companyId / ownership scoping), payments or money math, GL/journal posting, database migrations, secrets/env, file or row deletion, or session/cookie handling**. An ungated high-stakes item is executable only when **both** hold: (a) `--verify fresh` is in effect, and (b) the invoker explicitly listed it in `--only` — i.e. a human or the NEXT.md routing chose it for this run. Otherwise park it as `needs-human` (note: "high-stakes; route via /ratchet-recommend"). Exception: a `[BASELINE]` item is exempt from the `--only` requirement — its tag *is* the human routing, and the run cannot proceed without it — but use fresh verification for it when it touches the trigger list and fresh is available. No self-assessment about which model you are — the flags decide. A wrong "green" on auth or money is the most expensive failure Ratchet exists to prevent; the cost of a human glance is always lower.

## Per item — the inner loop

1. **Mark `in-progress` and set `Attempts` to 1 in the Ledger** (edit the file now, before any code). A crash must leave a visible trace, and the first implementation pass *is* attempt 1.
2. **Re-verify the Evidence.** Read the cited files; line numbers are anchors from authoring time and drift as earlier items in this same run edit the tree. **Match on the described pattern, not the line number** — grep for the code the Evidence describes; if it moved, use the new location; if it is gone or already fixed → Ledger `needs-human`, Journal one line explaining, next item. Never improvise a different interpretation of a stale item.
3. **Implement the Spec — nothing more.** Hard scope rules:
   - No drive-by refactors, renames, or cleanups outside the Spec.
   - No new dependencies unless the Spec names them.
   - Match surrounding code style; comments only for constraints the code can't show.
   - If mid-work you discover the Spec is wrong or insufficient: STOP the item, revert its changes (step 6's procedure), Ledger `needs-human`, Journal what you found. The Spec is the human's; you don't rewrite it silently.
4. **Verify:** run every Acceptance criterion, then every Global check. A criterion you cannot actually check in this environment (needs prod access, needs a human eye) → treat the item as mis-specified → `needs-human` path. **Never mark a criterion satisfied on inference — only on observation.**
5. **Red → diagnose and fix, bounded.** Each new fix cycle increments `Attempts` — the count is total implement+verify cycles, and the cap is 3 (a red verify at attempt 3 means block; a per-item Note from a human can override the cap). Between attempts, Journal one line: what failed, what you're changing. Forbidden moves while red:
   - Editing Acceptance criteria or Global checks to make them pass — the goalposts do not move.
   - Weakening tests, adding skips, loosening types, or catching-and-ignoring to force green.
   - Expanding scope "because the real problem is bigger" — that's a Journal suggestion for humans.
6. **Attempts exhausted → block cleanly.** Revert the item's *source* changes, but **never revert or delete the backlog file or its state directory** — that is the loop's own state and it must survive to record the block. So: `git restore --source=HEAD --worktree -- <the source paths this item touched>` (enumerate them; do **not** `git checkout -- .`, which would also wipe your in-progress ledger edits), then remove only the source files this item created — explicitly, by name. Avoid a blanket `git clean -fd`: if the state directory isn't committed yet it will delete your audit and backlog. Confirm `git status` shows only the backlog file modified (your ledger/journal writes), then set Ledger `blocked` + attempts count; Journal: what failed, best hypothesis, what a human should look at. Commit the ledger/journal update (`ratchet(<ID>): blocked — <reason>` or the repo's convention) so the block is durable — this commit also carries any outstanding sha backfill.
7. **Green → the ratchet clicks.** In this order:
   1. Update the item's Ledger row: `done`, final `Attempts` count, note, and `Commit: pending` — a commit cannot contain its own sha.
   2. Stage exactly the item's source paths plus the backlog file (`git add <src...> <backlog-path>`). Confirm the staged diff contains nothing else; the only extra hunk permitted is the *previous* item's sha backfill (`pending` → sha).
   3. Commit once. Message: the detected repo convention; fallback `ratchet(<ID>): <title>`. Never `--no-verify`; if a hook fails, that's a red verify — go to step 5.
   4. Read the new short sha (`git log -1 --format=%h`) and write it over this row's `pending`. That edit stays in the working tree and rides in the next ledger write — the next item's commit, a block-recording commit, or the run-ending `ratchet: sync ledger` commit.

## Verification modes

- **`inline`** (default) — you run the acceptance criteria and global checks yourself, and you may only mark them on observation, never on inference.
- **`fresh`** — after your own checks pass, spawn a **clean-room verification subagent** that receives ONLY: the item's text, its acceptance criteria, the global checks, and the repo path — *not* your implementation reasoning or diff narrative. It independently re-runs every criterion and reports pass/fail per criterion. Any disagreement counts as red (back to the fix loop; the subagent's failing evidence goes in the Journal). The point: after several red attempts, an implementer starts seeing green that isn't there — a verifier with no stake in the outcome doesn't. Costs more; use it for high-stakes items (`[P0]`, security, data), for anything NEXT.md routed SUPERVISED/SENIOR, and for unattended runs.

## Stop conditions — the outer loop ends when

- No eligible items remain (report: **backlog dry**).
- **Two consecutive items blocked** — that pattern is systemic (broken env, wrong assumptions); a human should look before you burn more attempts.
- `--max-items` reached.
- Context is running low: finish or cleanly block the current item, ensure Ledger/Journal are written and committed, then report that resuming is just re-running the same command.

**Whatever the stop reason:** before reporting, commit any outstanding ledger edits as `ratchet: sync ledger`. A run must end with a clean tree and no `pending` sha left in the file.

## Final report (always, even on early stop)

- Table: ID · terminal status · attempts · commit.
- Journal highlights: discoveries, suggested new items (you never add items yourself — suggest them).
- Parked items needing humans, grouped by tag — and for each, the exact edits that release it (the format's Human transitions: journal the decision, clear the gate tag, make provisional criteria real, status → `todo`, reset Attempts if it was `blocked`).
- Exact resume command.
- Pointer: suggest `/ratchet-recommend` to regenerate `NEXT.md` (in the state directory) — the routed plan for what to do next and who (autonomous cheap model / supervised / senior / human) should do each item now that the ledger has moved.

## Failure modes you must not exhibit

- **Acceptance theater** — running only the typecheck when the criteria list a behavior check.
- **Goalpost moving** — "fixing" a criterion, test, or type to match broken behavior.
- **Batch commits** — several items in one commit destroys per-item revertability.
- **Silent scope growth** — the most common way loops turn a codebase to soup.
- **Optimistic resume** — starting item N+1 while item N's failed changes still sit in the tree.
- **Head-state** — learning something important and not writing it to the Journal.
- **Committing to a push-to-deploy default branch without asking** — preflight step 7 exists for a reason.
- **Self-grading under pressure** — attempt 3, deadline energy, and suddenly the flaky check "basically passes". It doesn't. That failure mode is exactly what `--verify fresh` exists for.
