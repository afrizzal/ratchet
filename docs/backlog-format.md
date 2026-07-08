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
| `Acceptance:` | yes, ≥1 | Checkbox list. Each criterion is either a backticked command with its expected result, or an observable behavior a session can actually check. **No acceptance criteria, no item.** Vague criteria ("works well") are a validation error. |

Items are **immutable to executors**. Humans edit items; the loop only writes to the Ledger and Journal. In particular: an executor may never edit acceptance criteria — if a criterion is wrong or unverifiable, the item becomes `needs-human` with a journal note. The goalposts don't move.

## 3. Ledger

The single source of truth for state. One row per item, same order as Items.

```markdown
## Ledger
| ID | Status | Attempts | Commit | Note |
|---|---|---|---|---|
| SEC-01 | done | 2 | a1b2c3d | second attempt: also needed the list route |
| PERF-02 | todo | 0 | — | — |
| OPS-03 | needs-human | 0 | — | [OPS] requires prod .env access |
```

Status machine (executors may only follow these arrows):

```
todo → in-progress → done
                   → blocked        (attempts exhausted; item's changes reverted)
                   → needs-human    (human-gated tag, stale evidence, or wrong criteria)
todo → skipped                      (human decision only — executors never skip)
```

Rules:
- Mark `in-progress` **before** starting work (a crash then leaves a visible trace).
- `done` requires: item acceptance green + global checks green + an atomic commit that contains both the change **and** this ledger row update. The commit sha goes in the `Commit` column.
- `blocked` requires: the item's working-tree changes fully reverted, and a Journal entry saying what failed and the current hypothesis.

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

1. Refuse files without the `ratchet:v1` marker or with malformed items (missing Spec/Acceptance, duplicate IDs).
2. Eligible item = status `todo` + no human-gate tag + all `Depends:` done. Take them in file order (priority is expressed by ordering and the `[P*]` label humans assigned).
3. Edit only Ledger + Journal. Items are read-only.
4. One item = one commit = code change + ledger update together.
5. Stop conditions: no eligible items; two consecutive `blocked`; an explicit item/time budget from the invoker; or context exhaustion (write state, report how to resume).

Anything not specified here is executor freedom — but when in doubt, prefer the boring choice: it keeps ports compatible.
