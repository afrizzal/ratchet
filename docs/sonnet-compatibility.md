# Moved: Executor Rules

These rules now live **inside the skill that needs them**: [`skills/ratchet-loop/executor-rules.md`](../skills/ratchet-loop/executor-rules.md).

They were previously stranded here — installed executors copy only `skills/*`, so the cheap model
the rules were written for never saw them. `ratchet-loop`'s SKILL.md now instructs every executor to
load `executor-rules.md` from its own skill directory before touching an item.

If you linked to this file, update the link. The content is unchanged apart from two fixes:
rule 8 now defines `Attempts` as total implement+verify cycles (first pass = attempt 1), and
rule 11 documents the `Commit: pending` → sha-backfill mechanism (a commit cannot contain its own sha).
