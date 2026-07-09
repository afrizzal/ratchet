# Contributing to Ratchet

Thanks for turning the crank with us. Ground rules — short on purpose:

## What we'd love

- **Ports.** The backlog format is agent-agnostic by design. Cursor rules, Codex prompt packs, bare-script executors — open a PR adding a `ports/<name>/` folder with a README.
- **Field reports.** Ran the loop on a real repo? An issue titled `field report: <stack>, <n> items` with what worked/broke is worth more than code.
- **Sharper failure modes.** Each skill has a "failure modes" section. If the loop did something dumb on your repo, the fix is usually one more rule there.

## What needs an issue first

- **Changes to the backlog format** (`docs/backlog-format.md`). The format is the public contract — breaking it breaks every port. Propose in an issue first. Two live versions: **v1** (§1–4, single-writer) and **v2** (§5, parallel lanes). Neither may change in a way that invalidates a conforming file of the other; a v3 gets a new marker, not a redefined one.
- New skills. Ratchet stays a five-skill story (audit → backlog → recommend → loop → ship); new capabilities should usually extend an existing skill.

## Migrating a backlog from v1 to v2

v1 is not deprecated and stays the default: one agent, one file, zero coordination. Migrate when the backlog is big enough that parallelism beats simplicity — and when the work actually *partitions* (two areas of the tree that different items touch without overlapping). If every item touches `src/core/`, lanes buy you nothing.

The conversion is one command and one commit:

```
/ratchet-backlog migrate            # asks you for the partition, then rewrites in place
```

What it does, and what you should check in the diff:

1. **You choose the partition.** Lanes follow parallel-safe seams in the tree, not team names. Each named lane declares disjoint path globs; the first-declared lane is the **default lane**, its Scope is the literal `(rest)`, and it owns every path no named lane claims.
2. **Every item gains `Lane:`.** Items that add a dependency, edit shared config, or touch generated files belong in the default lane — its `(rest)` scope is what makes them legal, and it runs as a barrier so they can never collide with a parallel wave. *Lane assignment follows the diff an item will produce, not its topic.*
3. **Ledger rows move** verbatim into `lanes/<lane>.md`. IDs are never renumbered.
4. **Journal lines are copied** into the lane of every item ID they mention (whole-token match). A line naming two items lands in both — duplication is harmless in an append-only history, and losing a lesson is not.
5. `## Ledger` / `## Journal` leave `BACKLOG.md`; the marker flips to `<!-- ratchet:v2 -->`.

Preconditions and gotchas:

- **Finish the run first.** `migrate` refuses while any row is `in-progress`. An at-rest `pending` sha is not a blocker — it backfills it from `git log`.
- **Migrate on the integration branch**, with no lane branches in existence yet. After migrating, commit before launching any lane run.
- **`done` and `skipped` items are exempt from the scope checks.** History doesn't re-validate, so a finished item whose old Spec straddles your new partition never blocks the migration.
- **Ports:** a v2 executor must implement §5 in full — run markers, the barrier lane, the human write window. A port that reads lane files but skips run markers is not a v2 executor; it is a race.

Going back is just as cheap: concatenate the lane ledgers into one `## Ledger`, merge the journals by date, drop `## Lanes` and the `Lane:` fields, flip the marker. Nothing in v2 is one-way.

## House style

- Skills are **one folder** — `skills/<name>/SKILL.md`, plus at most one or two reference files the skill loads at runtime (e.g. `ratchet-loop/executor-rules.md`). English, imperative voice, no fluff. Anything a skill needs at runtime must live *inside its folder* — installed skills can't see `docs/`.
- Every rule in a skill must earn its place: it prevents a specific, nameable failure.
- Examples over adjectives.
- **Some contracts are mirrored across files and must be edited together.** The high-stakes trigger list is character-identical in `ratchet-loop/executor-rules.md` rule 1 (authoritative), the loop SKILL's high-stakes gate, and recommend's *Sensitive* definition — small models keyword-match it literally. Likewise the high-stakes gate itself (`--verify fresh` **and** explicit `--only`; `[BASELINE]` exempt from `--only` only), the `Commit: pending` sha-backfill mechanism, the 3-attempt cap, and the human-only unpark procedure each live in the format doc *and* two or three skills. Before shipping a change to one, grep the others.

## Testing a skill change

Copy the modified folder into `~/.claude/skills/`, run it against a scratch repo, and paste the transcript highlights into your PR description. There is no CI that can judge prompt quality — reviewers read transcripts.
