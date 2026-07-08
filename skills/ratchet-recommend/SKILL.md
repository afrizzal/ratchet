---
name: ratchet-recommend
description: Turn a ratchet BACKLOG.md + Ledger into a prioritized, model-routed execution plan (ratchet/NEXT.md) — what to do next, who should do each item (an autonomous cheap model, a supervised cheap model, a senior model, or a human decision), in what order, and with what verification. Read-only: changes no code, no items, no ledger. Use after /ratchet-audit, after a /ratchet-loop run, or whenever the user asks "what should I do next?".
---

# Ratchet Recommend — the navigator

Your output is a **route, not a report card**. The ledger already says *what state each item is in*; a report just re-states it. Your job is to say *what to do next, who should do it, and in what order* — so that a human, or a cheaper model, can act on the backlog without re-deriving the senior judgment that produced it. That routing **is** the thing that lets a small model land a fix as safely as an expensive one: it inherits the decision instead of guessing it.

## Invocation

```
/ratchet-recommend [path]
```
- `path` — backlog file. Default `ratchet/BACKLOG.md`. **Read-only.**

## What it reads / writes

- **Reads:** the Items (Tags, priority, Spec, Acceptance, `Depends:`), the Ledger (status per item), the Journal.
- **Writes:** only `ratchet/NEXT.md`. It changes no source, no item, no ledger row, no tag. It recommends; the loop executes and humans decide.

## Phase 1 — Snapshot

Count ledger states (`done` / `todo` / `needs-human` / `blocked` / `skipped`). Check the baseline: if a `[BASELINE]` item is not `done`, **nothing else matters** until it is — say so first.

## Phase 2 — Route every non-terminal item into a lane

Apply top-down; **first match wins**. The mapping is mechanical — derived only from the item's Tags, priority, `Depends:`, and the shape of its Acceptance — so a cheaper model reaches the same routing a senior would.

| Lane | Match | Who runs it | Verify |
|---|---|---|---|
| **BASELINE** | `[BASELINE]` while the baseline is red | anyone, **first** | its own AC |
| **DECIDE-FIRST** | `[USER-DECISION]` | a human answers a question; then it re-routes | — |
| **OPS** | `[OPS]` | human/ops (needs infra/credentials/a plan choice) | — |
| **SENIOR** | `[P0]`/HIGH **and** Spec touches auth · tenant boundary · money/GL · deletion · concurrency, **and** the change is non-trivial | strong model **+ `--verify fresh`**; never a cheap model solo | fresh |
| **MIGRATION** | `[RISKY]` whose Spec includes a schema/DB migration | supervised + human reviews the SQL, needs a DB/shadow-db | fresh |
| **SUPERVISED** | touches a sensitive area but is **small** (~<50 lines) and has a runnable AC | cheap model **allowed, with `--verify fresh`** | fresh |
| **AUTONOMOUS** | no gate tag, not `[P0]`, **every** AC is a runnable command, and it touches **none** of {auth, tenant, money/GL, migration, deletion, secrets} | cheap model, unattended loop | inline |

Guard rails on the mapping:
- **Never route an auth / money / tenant / migration / deletion item to AUTONOMOUS**, even if it looks tiny and its tests run — SUPERVISED is the floor for sensitive areas. A wrong "green" there is the most expensive failure Ratchet exists to prevent.
- If an item's Acceptance is **all** manual/infra ("visit in a browser", "EXPLAIN shows…", "`gh api`…") with no runnable command, route it DECIDE-FIRST with the note *"acceptance not machine-checkable — define a runnable criterion or accept manual sign-off."*

## Phase 3 — Order into waves

Respect `Depends:` (a lane can't start before its dependencies are `done`). Then order by leverage:

1. **BASELINE** — unblocks the loop itself.
2. **DECIDE-FIRST** — surface the questions **now**, even though they execute later: a human can answer them in parallel while other waves run. A plan that hides its blockers stalls.
3. **AUTONOMOUS** quick wins — cheap, safe, shrink the backlog, build momentum.
4. **SUPERVISED** (fresh-verify).
5. **SENIOR** — often the highest-**value** items (the security/money HIGHs) but the **least automatable**. Say that tension out loud and put them where a person will see them, not buried at the bottom.
6. **MIGRATION** / **OPS** — batched; need environment prep.

Within a wave, higher priority (`[P0]` → `[P3]`) first.

## Phase 4 — Write `ratchet/NEXT.md`

- **▶ Start here** — the single safest high-value action to take *right now*, with the exact command. If the top-value item is SENIOR (can't be automated), say both tracks: "schedule a human for X; meanwhile a cheap loop can clear Wave 1 with `<command>`."
- **Decisions needed from you** — each DECIDE-FIRST item written as a **concrete question** (the actual choice, not "needs a decision"). Note what it unblocks.
- **Waves** — the ordered plan. Per item: `ID · lane · who (autonomous/supervised/senior/human) · verify (inline/fresh) · one-line why · exact invocation` (e.g. `/ratchet-loop --only SEC-05,TEST-01 --verify inline`).
- **Do NOT automate** — the SENIOR list spelled out, each with the one-line reason a cheap model would get it wrong.
- **State** — the ledger counts, so progress is visible at a glance.

## Rules

- **Read-only.** You recommend; the loop executes; humans decide. Never edit tags, ledger, code, or acceptance.
- **Route mechanically** from tags + priority + acceptance shape — a reader must be able to reproduce your routing from the same inputs. No vibes.
- **Surface `[USER-DECISION]` items as real questions**, not "needs a decision".
- **Regenerate after every loop run** — the ledger moved, so the route moved.

## Failure modes to avoid

- A list that restates the ledger without saying *who* does each item and *in what order* — that's a report, not a recommendation.
- Routing a security/money HIGH into a cheap autonomous loop because its tests happen to be runnable.
- Burying the human-decision items so the plan silently waits on an answer nobody was asked.
- Recommending work whose `Depends:` aren't `done` yet.
