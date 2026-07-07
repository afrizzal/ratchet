# Executor Rules — binding for every model running the loop

Ratchet's promise is that a **sharp backlog lets a cheaper model execute safely what an expensive
model discovered.** These rules encode the judgment a senior model applies automatically, as
operational checks a smaller model (Sonnet) can follow literally. They were extracted from a field
test on a production multi-tenant ERP (Next.js/tRPC/Prisma, 40 routers, 76 models).

Each rule is a *do-this-exactly*, not a principle. When a rule and your instinct disagree, follow the rule.

---

## 1. Identify risky work by a fixed trigger list, not by feel
If an item's Spec or Evidence touches any of: **auth/permissions, tenant boundaries (companyId /
ownership scoping), payments or money math, GL/journal posting, database migrations, secrets/env,
file or row deletion, or session/cookie handling** — it is high-stakes. An ungated high-stakes item
is executable only when **both** hold: `--verify fresh` is in effect **and** the invoker explicitly
listed it in `--only` (a human or the NEXT.md routing chose it for this run). Otherwise set it
`needs-human` (note: "high-stakes; route via /ratchet-recommend") and continue with the next item.
This is the same gate the loop's SKILL.md states — one rule, two places, identical wording.
(Exception: a `[BASELINE]` item is exempt from the `--only` requirement — its tag *is* the human
routing — but still use fresh verification for it when it touches this list and fresh is available.)

## 2. Never widen scope past the Spec — the diff must be enumerable in one sentence
Implement only what the Spec names. Before committing, ask: *"can I describe this diff as one change?"*
If the honest answer contains "and also" (renamed a variable, tidied an import, fixed a nearby bug),
revert the "and also". A drive-by cleanup that isn't in the Spec is a bug in your commit, even if it's
an improvement. New dependencies are forbidden unless the Spec names them.

## 3. Judge evidence by opening the file, and grep the pattern not the line
Line numbers drift. Before editing, grep for the *code the Evidence describes* (the function name, the
string, the query shape) — not the cited line number. Three outcomes only: (a) pattern found where
expected → proceed; (b) pattern found elsewhere → use the new location; (c) pattern gone / already
fixed → `needs-human`, Journal one line, next item. Never "interpret what the item probably meant".

## 4. A criterion is checkable only if it is a command that returns an exit code
"Works well", "is clean", "user sees the right thing", "EXPLAIN shows an index scan", "visit in a
browser" are **not** things you can verify — they are opinions or need a human/infra you don't have.
Mark the item done only when every criterion is either a backticked command you ran to green, or an
observable you literally observed in output you produced. If an item's only criteria are un-runnable,
it is mis-specified → `needs-human`. **Never mark a criterion satisfied by inference.**

## 5. An item is too large if it can't be one reviewable commit
Signs it's actually two items: the Spec has numbered sub-steps that each change different files for
different reasons; it says "refactor X"; it touches a schema migration *and* application logic *and*
tests as three separate concerns. If you're 40 lines in and the end isn't in sight, stop, revert,
Journal "item is larger than one commit — suggest splitting into A/B", set `needs-human`. (A single
mechanical change repeated across N files — e.g. the same one-line fix in 8 routers — is still *one*
item; "large" is about number of distinct decisions, not number of lines.)

## 6. The goalposts are frozen — you may never edit acceptance criteria or tests to pass
When a check is red, you fix the code, never the check. Forbidden while red: editing an Acceptance
criterion or Global check; weakening/skipping a test; loosening a type; adding a try/catch that
swallows the failure; `--no-verify`. If a criterion is genuinely wrong or unverifiable, the item goes
to `needs-human` with a Journal note — you do not "correct" it. Attempt-3 deadline energy is exactly
when this rule matters most.

## 7. Preserve conventions by reading them first
Before your first edit, read `CLAUDE.md` / `CONTRIBUTING` / the surrounding file. Match the commit
message style from `git log` (this repo used Conventional Commits in Indonesian — so did every commit
the loop made). Use the project's existing helper (this ERP had a `DeleteConfirmDialog`, a
`getCompanyId(ctx)` scoping helper, a `postJournal` primitive) instead of inventing one. If the repo
bans a pattern (`confirm()`, `asChild`, `Float` for money), do not introduce it. When your fix and the
local style disagree, the local style wins.

## 8. Handle failing tests as diagnosis, bounded to 3 attempts total
`Attempts` counts implement+verify cycles: the first implementation pass is attempt 1; each further
fix cycle increments it. Red → read the actual failure, form one hypothesis, change one thing, re-run.
Journal one line per attempt (what failed, what you changed). A red verify at attempt 3: revert the
item's source changes only, set `blocked`, Journal the best hypothesis and what a human should look
at. Do not thrash past 3. Do not "sort of" pass — a flaky check that "basically works" has not passed.

## 9. Decide to revert on a fixed rule, and revert surgically
Revert the item when: attempts hit 3, OR you discover mid-work the Spec is wrong/insufficient, OR the
acceptance criteria can't be run here. Revert only the **source files this item touched** — enumerate
them by name and `git restore` them; delete only files this item created, by name. **Never**
`git checkout -- .` or `git clean -fd` — those wipe your ledger edits and can delete the untracked
`ratchet/` state. After reverting, `git status` should show only the backlog file changed.

## 10. Update the Journal so the next session inherits, not re-discovers
One line per meaningful event: attempt result, block reason, a discovery worth keeping ("the ledger
tables are FK-less, so a delete won't be blocked by the DB"). Write what a fresh session with zero
memory of this run would need. If you learned something that suggests a *new* item, write it as a
Journal suggestion — you never add items yourself. Head-state that isn't in the file is lost the moment
your context resets.

## 11. Produce reviewable commits: one item, two-file diff shape
A finished item is exactly one commit containing (a) the item's source change and (b) the backlog's
ledger row flipped to `done` with `Commit: pending` (a commit cannot contain its own sha). The **only**
other hunk permitted in the diff is the one-line sha backfill of the *previous* item's row
(`pending` → short sha) — nothing else. Stage the specific paths (`git add <src...> <backlog-path>`,
where `<backlog-path>` is the backlog file resolved at preflight — default `ratchet/BACKLOG.md`),
confirm the staged diff is only those files, then commit with the repo's message convention
(`ratchet(<ID>): <title>` if none detected). After committing, write the new short sha over your own
row's `pending`; that edit rides in the next commit. Never batch several items into one commit —
per-item revertability is the whole point.

## 12. Stop instead of improvising
Stop and hand back to a human when: two items block in a row (something systemic is wrong), or
context runs low (write state, commit, report the resume command). Everything item-shaped — a Spec
that contradicts the code, a rule-1 or rule-4 trip, stale evidence — is *parked* (`needs-human`)
and the loop continues. Parking is per-item; stopping is for run-level problems. Before stopping — for any reason — commit outstanding ledger edits as
`ratchet: sync ledger` so no `pending` sha and no dirty tree survive the run. Stopping cleanly with
the state written is a *success*. Improvising past an ambiguity — guessing what a stale item meant,
forcing a red check green, expanding scope "because the real problem is bigger" — is the failure mode
Ratchet exists to prevent. When unsure, park it.

---

### The one-line version
*Implement exactly the Spec; verify by running, never by believing; freeze the goalposts; and when the
map stops matching the territory, stop and write down where you are.*
