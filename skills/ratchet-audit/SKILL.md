---
name: ratchet-audit
description: Deep multi-agent audit of the current codebase that ends in an executable artifact — ratchet/AUDIT.md (understanding + findings) and ratchet/BACKLOG.md (prioritized items with evidence, minimal specs, and runnable acceptance criteria, in ratchet:v1 format). Use when the user says "audit this repo", "ratchet audit", "review the codebase and give me a backlog", or before pointing /ratchet-loop at a project for the first time. Read-only: changes no source files.
---

# Ratchet Audit — from codebase to executable backlog

Your output is not a report card; it is **work orders**. Every finding must survive verification down to file:line and arrive with acceptance criteria a later session can actually run. A finding that can't be turned into a checkable item belongs in AUDIT.md's notes, not in the backlog.

## Invocation

```
/ratchet-audit [focus] [--depth quick|standard|deep]
```

- `focus` — optional: `security`, `performance`, `tests`, `data`, `frontend`, or free text ("multi-tenant isolation"). Default: full sweep.
- `--depth` — `quick` (no subagents, top risks only), `standard` (default), `deep` (more agents, adversarial verification pass).

## Phase 1 — Fingerprint the project (yourself, fast)

1. Read the manifests and configs: `package.json` / `pyproject.toml` / `go.mod` / `Cargo.toml`, lockfiles, CI workflows, Dockerfiles, env examples, top-level README/CLAUDE.md/CONTRIBUTING.
2. Determine: stack, entry points, how it's tested (**capture the exact commands** — they become Global checks), how it's deployed, and the domain axes that matter (multi-tenant? auth? payments? background jobs?).
3. **Baseline reality check (mandatory).** Run each candidate Global check **cold** — fresh clone semantics: no incremental cache, no dev-server-generated types, the way CI runs it. A check that passes warm but fails cold is a *non-deterministic* check and is worse than useless as a gate (the loop will read a false green). When you find one: (a) pin the Global check to its deterministic form (e.g. `tsc --noEmit --incremental false`, not `tsc --noEmit`), and (b) if the cold run is **red**, that is not a "skip" — it is your **first backlog item**, tagged `[BASELINE]`, because the loop cannot ratchet forward from a red baseline. Never emit a backlog whose own Global checks are red with no `[BASELINE]` item to fix them.
4. Pick audit dimensions by project shape — typical for a web app: **data layer/schema · API surface/authz · business logic/duplication/fragility · tests/CI/ops · frontend/UX debt**. A CLI or library gets different cuts (public API, error handling, packaging, docs). Don't force irrelevant dimensions.

## Phase 2 — Fan out (parallel subagents; skip at `--depth quick`)

Launch one read-only exploration agent per dimension, **in parallel**. Each agent's brief must demand:
- Facts with `file:line` citations, never file dumps.
- Explicit separation of **observed** vs **inferred**.
- A ranked list of concrete defects/risks for its dimension, each with: what, where, why it matters, and how one would *prove* it's fixed (this becomes the acceptance criteria seed).
- Counting where it strengthens claims (route totals, models without a tenant column, dead files with zero importers).

Prompts must be self-contained (agents don't see your conversation): include the repo path, the stack fingerprint from Phase 1, and the dimension's specific questions.

## Phase 3 — Verify before you write

For every candidate finding: open the cited file yourself and confirm the evidence holds **today**. At `--depth deep`, additionally spawn independent verification agents prompted to **refute** each HIGH/MED finding (each verifier gets only the claim + repo path, not the finder's reasoning); a finding survives only if the refuter fails. Drop or downgrade anything you cannot confirm — the loop downstream will trust your evidence literally, so unverified findings poison everything after them. Severity honestly: HIGH = exploitable/corrupting/money-losing now · MED = will bite under a plausible condition · LOW = debt and drift. Never inflate.

## Phase 4 — Write the two artifacts

Create the `ratchet/` directory (respect an existing one: **append/update, never clobber** a BACKLOG.md that has Ledger progress — new items go under new IDs).

**If the existing backlog is v2** (marker `<!-- ratchet:v2 -->`; items carry `Lane:`, state lives in `lanes/<lane>.md`): append items **with** a `Lane:` field — pick the declared lane whose scope contains the item's Evidence/Spec paths; dependency-adding or shared-path items go to the first-declared default lane; if no assignment is defensible, the finding goes to AUDIT.md's "needs definition" list instead. Write each new item's `todo` ledger row into its lane's file, **never** emit `## Ledger`/`## Journal` sections into a v2 backlog, and touch nothing while a lane run is open (a lane journal whose last run marker is `run started` — respect the format's human write windows; if every lane is open, put the items in AUDIT.md and say so). Fresh emission stays v1 below — v2 is opt-in via `/ratchet-backlog migrate` or an explicit user ask.

**`ratchet/AUDIT.md`** — the context: what the project is, architecture, hidden assumptions, what's strong and must be preserved, what's fragile, what remains unclear (say exactly what you couldn't verify — never bluff).

**`ratchet/BACKLOG.md`** — a **fresh** backlog is always `ratchet:v1` (v2's parallel lanes are opt-in, via `/ratchet-backlog migrate` or an explicit user ask; appending to an existing v2 file follows the rule above). The skeleton you must emit (full spec: `docs/backlog-format.md` in the ratchet repo, github.com/afrizzal/ratchet):

```markdown
# BACKLOG — <project or scope name>
<!-- ratchet:v1 -->

## Global checks
- `<verify command, deterministic form>`

## Items

### SEC-01 — <title> [P0]
- Tags: —                  ← only [RISKY] [OPS] [USER-DECISION] [BASELINE], or —
- Depends: —
- Evidence: path:line — <verified observation>
- Spec: <minimal change, one commit>
- Acceptance:
  - [ ] `<runnable command>` <expected result>

## Ledger
| ID | Status | Attempts | Commit | Note |
|---|---|---|---|---|
| SEC-01 | todo | 0 | — | — |

## Journal
```

Field rules:
- `## Global checks` = the project's real verify commands captured in Phase 1.
- Items sized to **one commit each** — split anything bigger.
- ID prefixes by category (`SEC-`, `PERF-`, `TEST-`, `DX-`, `DOC-`...), ordered by priority.
- Evidence = the verified `file:line — observation`.
- Spec = the minimal change, written so a *smaller model* can execute it without creativity.
- Acceptance = runnable commands + observable behaviors. If you can't write a checkable criterion, the item isn't ready — move it to AUDIT.md's "needs definition" list instead.
- Tag honestly, using **only** the canonical gate vocabulary in the `Tags:` field: `[RISKY]` (migration/destructive/irreversible/wide blast radius), `[OPS]` (needs credentials/infra), `[USER-DECISION]` (product/policy judgment), `[BASELINE]` (fixes a red baseline). **Normalize** — subagents return free-form tags (`auth`, `idor`, `perf`); those are *descriptions*, not gates. Map each to a bracket tag or drop it. A finding tagged `privilege-escalation` but written into the file without `[RISKY]` reads as far safer than it is — this is the single most dangerous authoring mistake. When a finding touches auth, tenant boundaries, payments, migrations, secrets, deletion, or GL/money **and** is HIGH/P0, prefer the gate even if it is locally testable — a cheap executor implementing it wrong corrupts data that tests may not catch. (Ungated sensitive items are not defenseless — the loop's high-stakes gate still demands explicit `--only` routing plus `--verify fresh` before executing them — but the bracket tag is what forces a human review first, and if you have not seen the acceptance criteria pass in a real run, the tag is the right call.)
- Full Ledger (all `todo`, attempts 0) + empty Journal.

## Phase 5 — Report

Summarize to the user: top findings by severity (plain language, with the one-sentence exploit/failure story each), what's strong, item count by priority, and the handoff line: *review/edit `ratchet/BACKLOG.md`, then run `/ratchet-recommend` for a routed plan (who does what, in what order), or `/ratchet-loop` to start executing*.

## Rules

- **Read-only.** You change no source files; your only writes are the two artifacts.
- Every backlog item traces to verified evidence. No "consider improving X" items.
- Don't drown the signal: a 20-item backlog that's all real beats 60 items of filler. Long-tail debt goes in AUDIT.md prose.
- If the repo already has conventions docs (CLAUDE.md, CONTRIBUTING), audit *against* them and cite them in specs.

## Failure modes to avoid

- Unverified findings copied from agent summaries straight into items.
- Acceptance criteria the executor can't run ("verify in production").
- Mega-items ("refactor the auth system") — unshippable in one commit, guaranteed to block.
- Overwriting a backlog that has Ledger history.
- Generic advice untethered to this codebase.
- **Emitting a backlog with a red or non-deterministic baseline and no `[BASELINE]` item** — the loop stops on the first preflight and the whole artifact is dead on arrival.
- **Free-form tags in the `Tags:` field** — the loop only recognizes the four bracket gates; anything else reads as ungated.
- **Trusting subagent summaries verbatim.** Finders overclaim (this field-test's currency finding was MED until a verifier proved the harm path didn't exist yet → LOW) and mis-count (a brief said "114 models"; the schema had 76). Open the cited line yourself; at `--depth deep`, the refute pass is what catches inflation.
