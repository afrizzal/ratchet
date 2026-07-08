---
name: ratchet-loop
description: Execute a ratchet BACKLOG.md in a verified engineering loop — pick the next eligible item, implement exactly its spec, run its acceptance criteria until green (bounded attempts), lock it in as one atomic commit, update the ledger, repeat until the backlog is dry or a stop condition fires. Use when the user says "run the backlog", "ratchet", "work through BACKLOG.md", or wants autonomous-but-verified execution of a prepared item list. Requires a backlog conforming to the ratchet:v1 format.
---

# Ratchet Loop — the executor

You are executing a ratchet backlog. Your contract is **forward motion only**: an item is either done-verified-committed, or it is not done and the Ledger says exactly why. Nothing in between survives your run.

The backlog file is the single source of state. You are stateless; treat every run as if a different agent did the previous one (often true). Everything you learn that matters goes in the file, not in your head.

## Invocation

```
/ratchet-loop [path] [--max-items N] [--only ID[,ID...]] [--dry-run]
```

- `path` — backlog file. Default: `ratchet/BACKLOG.md`, else `BACKLOG.md` at repo root. If neither exists, say so and suggest `/ratchet-audit` or `/ratchet-backlog create`.
- `--max-items N` — stop after N items reach a terminal state this run.
- `--only` — restrict eligibility to the listed IDs (dependencies still respected).
- `--dry-run` — do phases 0–1 only: validate, report the execution plan, touch nothing.

## Phase 0 — Preflight (never skip, never reorder)

1. **Read the backlog file in full.** Verify the `<!-- ratchet:v1 -->` marker and validate structure per `docs/backlog-format.md` (unique IDs, every item has Spec + ≥1 Acceptance criterion, Ledger row per item). Malformed → STOP, report the exact problems, suggest `/ratchet-backlog validate`. Never guess at a malformed file.
2. **Require a clean working tree.** `git status` must show no staged or unstaged changes to tracked files. Dirty → STOP and ask the user (their work is not yours to stash). This invariant is what makes your failure-recovery safe.
3. **Read the Ledger and the entire Journal.** The Journal is your inheritance — previous attempts, known traps, failed hypotheses. Do not re-attempt something the Journal says failed without a new reason.
4. **Establish the green baseline.** Run every command under `## Global checks` once. Red baseline → STOP and report (you cannot ratchet forward from a broken start) — unless a `[BASELINE]`-tagged item exists, in which case that item is your mandatory first pick.
5. **Detect the repo's commit convention** from `git log --oneline -15` (conventional commits? plain sentences? scope prefixes?). You will match it.
6. **Announce the plan:** eligible items in order, parked items with reasons, the stop conditions in effect. In `--dry-run`, stop here.

## Eligibility

An item is eligible when ALL hold: Ledger status `todo` · no human-gate tag (`[USER-DECISION]`, `[OPS]`, `[RISKY]`) · every `Depends:` ID is `done` · matches `--only` if given. Take eligible items in file order. Human-gated `todo` items: set their Ledger status to `needs-human` with the tag as the note (once), and move on.

## Per item — the inner loop

1. **Mark `in-progress` in the Ledger** (edit the file now, before any code). A crash must leave a visible trace.
2. **Re-verify the Evidence.** Read the cited files; line numbers are anchors from authoring time and may have drifted. If the evidence no longer holds (code changed, already fixed, file gone) → Ledger `needs-human`, Journal one line explaining, next item. Never improvise a different interpretation of a stale item.
3. **Implement the Spec — nothing more.** Hard scope rules:
   - No drive-by refactors, renames, or cleanups outside the Spec.
   - No new dependencies unless the Spec names them.
   - Match surrounding code style; comments only for constraints the code can't show.
   - If mid-work you discover the Spec is wrong or insufficient: STOP the item, revert its changes (step 6's procedure), Ledger `needs-human`, Journal what you found. The Spec is the human's; you don't rewrite it silently.
4. **Verify:** run every Acceptance criterion, then every Global check. A criterion you cannot actually check in this environment (needs prod access, needs a human eye) → treat the item as mis-specified → `needs-human` path. **Never mark a criterion satisfied on inference — only on observation.**
5. **Red → diagnose and fix, bounded.** Increment `Attempts` in the Ledger each cycle. Cap: 3 attempts (or a per-item note from a human overriding it). Between attempts, Journal one line: what failed, what you're changing. Forbidden moves while red:
   - Editing Acceptance criteria or Global checks to make them pass — the goalposts do not move.
   - Weakening tests, adding skips, loosening types, or catching-and-ignoring to force green.
   - Expanding scope "because the real problem is bigger" — that's a Journal suggestion for humans.
6. **Attempts exhausted → block cleanly.** Revert everything this item touched: `git checkout -- .` then remove files the item created (you know them; you created them — remove explicitly, or `git clean -fd` which is safe *only because* Phase 0 guaranteed a clean start and every prior item committed). Confirm `git status` is clean. Ledger `blocked` + attempts count; Journal: what failed, best hypothesis, what a human should look at.
7. **Green → the ratchet clicks.** One atomic commit containing (a) the item's code changes and (b) the backlog file's Ledger row update (`done`, attempts, sha placeholder → fill after commit with the short sha via a tiny amend-free follow-up edit? No —) — do it in this order: update the Ledger row to `done` with attempts and note, stage code + backlog together, commit once, then write the short sha into the row and include that edit in the NEXT item's commit (or a final `ratchet: sync ledger` commit at run end). Commit message: follow the detected repo convention; fallback `ratchet(<ID>): <title>`. Never `--no-verify`; if a hook fails, that's a red verify — go to step 5.

## Stop conditions — the outer loop ends when

- No eligible items remain (report: **backlog dry**).
- **Two consecutive items blocked** — that pattern is systemic (broken env, wrong assumptions); a human should look before you burn more attempts.
- `--max-items` reached.
- Context is running low: finish or cleanly block the current item, ensure Ledger/Journal are written and committed, then report that resuming is just re-running the same command.

## Final report (always, even on early stop)

- Table: ID · terminal status · attempts · commit.
- Journal highlights: discoveries, suggested new items (you never add items yourself — suggest them).
- Parked items needing humans, grouped by tag.
- Exact resume command.

## Failure modes you must not exhibit

- **Acceptance theater** — running only the typecheck when the criteria list a behavior check.
- **Goalpost moving** — "fixing" a criterion, test, or type to match broken behavior.
- **Batch commits** — several items in one commit destroys per-item revertability.
- **Silent scope growth** — the most common way loops turn a codebase to soup.
- **Optimistic resume** — starting item N+1 while item N's failed changes still sit in the tree.
- **Head-state** — learning something important and not writing it to the Journal.
