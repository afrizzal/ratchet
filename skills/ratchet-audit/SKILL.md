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
3. Pick audit dimensions by project shape — typical for a web app: **data layer/schema · API surface/authz · business logic/duplication/fragility · tests/CI/ops · frontend/UX debt**. A CLI or library gets different cuts (public API, error handling, packaging, docs). Don't force irrelevant dimensions.

## Phase 2 — Fan out (parallel subagents; skip at `--depth quick`)

Launch one read-only exploration agent per dimension, **in parallel**. Each agent's brief must demand:
- Facts with `file:line` citations, never file dumps.
- Explicit separation of **observed** vs **inferred**.
- A ranked list of concrete defects/risks for its dimension, each with: what, where, why it matters, and how one would *prove* it's fixed (this becomes the acceptance criteria seed).
- Counting where it strengthens claims (route totals, models without a tenant column, dead files with zero importers).

Prompts must be self-contained (agents don't see your conversation): include the repo path, the stack fingerprint from Phase 1, and the dimension's specific questions.

## Phase 3 — Verify before you write

For every candidate finding: open the cited file yourself (or via a verification agent at `--depth deep`) and confirm the evidence holds **today**. Drop or downgrade anything you cannot confirm. Severity honestly: HIGH = exploitable/corrupting/money-losing now · MED = will bite under a plausible condition · LOW = debt and drift. Never inflate.

## Phase 4 — Write the two artifacts

Create the `ratchet/` directory (respect an existing one: **append/update, never clobber** a BACKLOG.md that has Ledger progress — new items go under new IDs).

**`ratchet/AUDIT.md`** — the context: what the project is, architecture, hidden assumptions, what's strong and must be preserved, what's fragile, what remains unclear (say exactly what you couldn't verify — never bluff).

**`ratchet/BACKLOG.md`** — strictly `ratchet:v1` (see `docs/backlog-format.md` in the ratchet repo, or mirror `examples/BACKLOG.example.md`):
- `## Global checks` = the project's real verify commands captured in Phase 1.
- Items sized to **one commit each** — split anything bigger.
- ID prefixes by category (`SEC-`, `PERF-`, `TEST-`, `DX-`, `DOC-`...), ordered by priority.
- Evidence = the verified `file:line — observation`.
- Spec = the minimal change, written so a *smaller model* can execute it without creativity.
- Acceptance = runnable commands + observable behaviors. If you can't write a checkable criterion, the item isn't ready — move it to AUDIT.md's "needs definition" list instead.
- Tag honestly: migrations/destructive → `[RISKY]`; needs credentials/infra → `[OPS]`; product judgment → `[USER-DECISION]`.
- Full Ledger (all `todo`, attempts 0) + empty Journal.

## Phase 5 — Report

Summarize to the user: top findings by severity (plain language, with the one-sentence exploit/failure story each), what's strong, item count by priority, and the handoff line: *review/edit `ratchet/BACKLOG.md`, then run `/ratchet-loop`*.

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
