---
name: ratchet-ship
description: Release runbook that takes verified local work to a green deploy — full local preflight, diff and secret review, push per the repo's convention, watch CI to completion, run the smoke checklist, and leave rollback notes. Use when the user says "ship it", "ratchet ship", "release", or after /ratchet-loop finishes a batch and the work should go out.
---

# Ratchet Ship — land it green

Shipping is the only ratchet click users feel. Your contract: nothing leaves the machine unverified, nothing merges while CI is red, and if it goes wrong there is a written way back.

## Invocation

```
/ratchet-ship ["summary of what's going out"]
```

## Phase 1 — Know the pipeline before touching it

1. Read the repo's release reality: CI workflows (`.github/workflows/`, GitLab CI, etc.), deploy docs (README/DEPLOYMENT/CLAUDE.md), branch model (`git log --graph --oneline -20`, default branch, PR-based or push-to-deploy?).
2. **If pushing the default branch triggers a production deploy, say so out loud before anything else** and get explicit confirmation. Push-to-deploy repos get no casual pushes.
3. Detect the local verify commands (test/typecheck/lint/build) from the manifest — and confirm each **actually runs**. A `lint` script that shells to a removed subcommand, or a `typecheck` that only passes with a warm cache, is a broken gate wearing a green coat; verify cold.
4. **Confirm the CI gate is enforceable, not just present.** A green pipeline that cannot block a merge is advisory theater. Check branch protection / required-status-checks actually exist (`gh api repos/{owner}/{repo}/branches/{branch}/protection`); if it returns 403/empty (e.g. a private free-plan repo), say so — red commits can and do reach the default branch, so your local gate is the real gate.

## Phase 2 — Preflight (local, all of it)

1. Run the full local gate: typecheck, tests, lint, build — whatever exists. Any red → STOP and report; shipping red is not your call to make.
2. Review the outgoing diff (`git diff <base>...HEAD --stat` + read the risky hunks):
   - **Secret scan** the diff: keys, tokens, passwords, connection strings, `.env` content. Hit → STOP immediately, tell the user (rotation may be needed if it was ever committed).
   - Large/binary files, lockfile churn without dependency intent, debug leftovers (`console.log`, commented-out blocks, skipped tests).
   - **Reproducibility of the CI build.** If CI deletes or regenerates the lockfile and runs `npm install` (not `npm ci`) — or otherwise floats dependency versions — the thing CI tested is not pinned to what you reviewed. Flag it; a green run on unpinned deps is a weaker signal than it looks.
3. Docs sync: if the repo maintains a CHANGELOG / task ledger / ratchet BACKLOG.md, confirm it reflects what's shipping; update if the convention demands it.
4. Migrations going out? Confirm they're additive or explicitly approved, and that a pre-deploy backup exists in the pipeline (or tell the user it doesn't — that's a finding, not a blocker you silently absorb).

## Phase 3 — Push, per the repo's own convention

- PR-based repo: branch → push → open PR with a body summarizing *what and why* (never paste raw diffs), following any PR template.
- Push-to-deploy repo: only after the Phase 1 confirmation. Atomic, conventional commit messages; never `--force` on shared branches; never `--no-verify` (a failing hook is a red gate, not an obstacle).

## Phase 4 — Watch it land

1. Follow CI to completion (`gh run watch` or equivalent). Do not declare success while anything is queued or running.
2. Red CI → read the actual failing step's log, diagnose, and either fix-forward (small, obvious, in-scope) or recommend rollback — with the failing log excerpt in your report either way.
3. Post-deploy smoke: if the repo defines a checklist (docs or `ratchet/SHIP.md`), run it; otherwise do the minimum honest check — the deployed thing responds, the changed surface works once, logs aren't screaming.

## Phase 5 — Report + rollback notes

Report: what shipped (commits/PR), gate results (each ✅/❌ with evidence), smoke results, and **rollback instructions specific to this deploy** (previous good ref/tag/image, the exact command, where backups live). If anything was skipped, say so plainly — a skipped gate reported is recoverable; a skipped gate hidden is how outages get long.

## Failure modes to avoid

- Declaring victory at "pushed" — the job ends at *green CI + smoke passed*, not at `git push`.
- Shipping with a red or skipped local gate "because CI will catch it".
- Quietly absorbing scary findings (secrets in history, missing backups) instead of surfacing them.
- Force-pushing or hook-skipping to make a gate stop complaining.
- Writing rollback notes generic enough to be useless ("revert the commit").
