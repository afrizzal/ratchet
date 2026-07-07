---
name: ratchet-recommend
description: Turn a ratchet BACKLOG.md + Ledger into a prioritized, model-routed execution plan (ratchet/NEXT.md) — what to do next, who should do each item (an autonomous cheap model, a supervised cheap model, a senior model, or a human decision), in what order, and with what verification. Read-only. Changes no code, no items, no ledger. Use after /ratchet-audit, after a /ratchet-loop run, or when the user asks "what should I do next?" about a project that has a ratchet backlog.
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
- **Writes:** only `NEXT.md`, alongside the backlog file (default `ratchet/NEXT.md`). It changes no source, no item, no ledger row, no tag. It recommends; the loop executes and humans decide.

## Phase 1 — Snapshot

Count all six ledger states (`done` / `todo` / `in-progress` / `needs-human` / `blocked` / `skipped`). An `in-progress` row with no run in flight is debris from a crashed session — the loop's preflight resets it to `todo` automatically, so route it as `todo` and note the crash. Check the baseline: if a `[BASELINE]` item is not `done`, **nothing else matters** until it is — say so first.

## Phase 2 — Route every open item into a lane

Route every item whose status is **not** `done` or `skipped` — that includes `needs-human` and `blocked`; their route says what releases them. The mapping is mechanical — derived only from the item's Tags, status, `Depends:`, and the literal text of its Spec/Evidence/Acceptance — so a cheaper model reaches the same routing a senior would. Two defined terms, both checkable from the file:

- **Sensitive** — the Spec or Evidence mentions any of the trigger list (verbatim from executor rule 1): **auth/permissions, tenant boundaries (companyId / ownership scoping), payments or money math, GL/journal posting, database migrations, secrets/env, file or row deletion, or session/cookie handling**.
- **Small** — the Spec names at most 2 files and no migration. When unsure, it is **not** small.

Routing order:

1. **Status overrides first.** `blocked` and `needs-human` items are governed by the status bullets below the table — those bullets pick the lane (sometimes by delegating back to the table) **and add the mandatory human release edits**. Only `todo` items (and crash-debris `in-progress`, treated as `todo`) enter the table directly.
2. **Then the table, top-down; first match wins.**

| Lane | Match | Who runs it | Verify |
|---|---|---|---|
| **BASELINE** | `[BASELINE]` and its Ledger row is not `done` (redness is judged from the Ledger, never by running commands). A human-gate tag **wins over** `[BASELINE]` — such an item routes to its gate row, flagged "baseline-blocking — unpark first" | anyone, **first**. If the item is *sensitive*, its invocation must carry `--only <ID> --verify fresh` (the loop exempts `[BASELINE]` from the `--only` requirement, but prescribe it anyway) | its own AC (fresh if sensitive) |
| **DECIDE-FIRST** | `[USER-DECISION]`, **or** every Acceptance criterion is manual/infra with no runnable command | a human answers the question / defines a runnable criterion; then unpark and re-route | — |
| **OPS** | `[OPS]` | human/ops (needs infra/credentials/a plan choice) | — |
| **RISKY-REVIEW** | `[RISKY]` (migration or not) | a human reviews the Spec — for migrations: the SQL plus a backup/shadow-db plan — then **unparks it** (removes the tag, journals the approval, status → `todo`, per the format's Human transitions); once unparked it re-routes through the rows below | per re-route — but the **first run after a `[RISKY]` unpark always carries `--verify fresh`**, whatever row it lands in |
| **SENIOR** | sensitive **and** not small | strong model **+ `--verify fresh`**, invoked with explicit `--only`; never a cheap model solo | fresh |
| **SUPERVISED** | sensitive **and** small | cheap model **allowed, with `--verify fresh`** and explicit `--only` | fresh |
| **AUTONOMOUS** | not sensitive **and** every AC is a runnable backticked command or an observable the executor can check from output it produces itself (the loop's own step-4 standard) | cheap model, unattended loop | inline |
| **SUPERVISED** (catch-all) | anything not matched above | cheap model with `--verify fresh` — the safe floor; if the item is large, recommend splitting it first | fresh |

Status bullets (these **override** the table):
- **`blocked` items** route to SENIOR (or DECIDE-FIRST if the Journal shows the Spec itself is wrong), quoting the loop's blocking hypothesis from the Journal. Their plan entry **must** include the release edits before the invocation: decision journaled, Attempts reset, status → `todo`.
- **`needs-human` items** route by the reason they parked. Gate-tagged → their tag's row above. Ungated parks — the high-stakes gate, "too large", "spec wrong", stale evidence, unrunnable criteria — route via the table rows, and like `blocked` items their plan entry **must** include the human release edits (journal the decision, fix what parked it, status → `todo`) before the invocation; a pasted command against a still-parked item does nothing.

Guard rail: **never route a sensitive item to AUTONOMOUS**, even if it looks tiny and its tests run — SUPERVISED is the floor, and the loop's own high-stakes gate enforces it (ungated sensitive items execute only with `--verify fresh` + explicit `--only`). A wrong "green" there is the most expensive failure Ratchet exists to prevent.

## Phase 3 — Order into waves

Respect `Depends:` (a lane can't start before its dependencies are `done`). Then order by leverage:

1. **BASELINE** — unblocks the loop itself.
2. **DECIDE-FIRST** — surface the questions **now**, even though they execute later: a human can answer them in parallel while other waves run. A plan that hides its blockers stalls.
3. **AUTONOMOUS** quick wins — cheap, safe, shrink the backlog, build momentum.
4. **SUPERVISED** (fresh-verify).
5. **SENIOR** — often the highest-**value** items (the security/money ones) but the **least automatable**. Say that tension out loud and put them where a person will see them, not buried at the bottom.
6. **RISKY-REVIEW** / **OPS** — batched; need a human's review or environment prep before anything can run.

Within a wave, higher priority (`[P0]` → `[P3]`) first.

## Phase 4 — Write `ratchet/NEXT.md`

Open with one plain sentence: **the loop does not execute this file — you do.** Each wave gives the exact command to paste; run them in order. (The loop *does* read NEXT.md when present and will refuse to silently downgrade a SUPERVISED/SENIOR item's fresh-verify — but the sequencing is yours.)

- **▶ Start here** — the single safest high-value action to take *right now*, with the exact command. If the top-value item is SENIOR (needs a strong model), say both tracks: "run wave N with a senior model; meanwhile a cheap loop can clear Wave 1 with `<command>`."
- **Decisions needed from you** — each DECIDE-FIRST item written as a **concrete question** (the actual choice, not "needs a decision"). Note what it unblocks and the unpark edits once answered.
- **Waves** — the ordered plan. Per item: `ID · lane · who · verify · one-line why · exact invocation` (e.g. `/ratchet-loop --only SEC-05,TEST-01 --verify fresh`). The **who** column names a concrete tier, not a vibe: *cheap model* = e.g. Sonnet (`/model sonnet`, or `claude --model sonnet` for a fresh session); *senior model* = the strongest model available (e.g. Opus/Fable); *human* = a person. SUPERVISED and SENIOR invocations always carry `--verify fresh` and an explicit `--only` list — that pair is what the loop's high-stakes gate checks.
- **Do NOT automate** — the SENIOR and RISKY-REVIEW lists spelled out, each with the one-line reason a cheap model would get it wrong, and for `[RISKY]` items the unpark procedure (review → remove tag → journal → `todo`).
- **State** — the ledger counts, so progress is visible at a glance.

A worked example ships with the ratchet repo at `examples/NEXT.example.md`.

## Rules

- **Read-only.** You recommend; the loop executes; humans decide. Never edit tags, ledger, code, or acceptance.
- **Route mechanically** from tags + status + the defined *sensitive*/*small* tests — a reader must be able to reproduce your routing from the same file. No vibes, and no inputs the file doesn't contain.
- **Surface `[USER-DECISION]` items as real questions**, not "needs a decision".
- **Regenerate after every loop run** — the ledger moved, so the route moved. (The loop's final report reminds the user.)

## Failure modes to avoid

- A list that restates the ledger without saying *who* does each item and *in what order* — that's a report, not a recommendation.
- Routing a security/money item into a cheap autonomous loop because its tests happen to be runnable.
- Burying the human-decision items so the plan silently waits on an answer nobody was asked.
- Recommending work whose `Depends:` aren't `done` yet.
- Emitting an invocation the loop will refuse — a `[RISKY]` item without its unpark step, or a SUPERVISED item without `--verify fresh`.
