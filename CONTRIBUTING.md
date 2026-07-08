# Contributing to Ratchet

Thanks for turning the crank with us. Ground rules — short on purpose:

## What we'd love

- **Ports.** The backlog format is agent-agnostic by design. Cursor rules, Codex prompt packs, bare-script executors — open a PR adding a `ports/<name>/` folder with a README.
- **Field reports.** Ran the loop on a real repo? An issue titled `field report: <stack>, <n> items` with what worked/broke is worth more than code.
- **Sharper failure modes.** Each skill has a "failure modes" section. If the loop did something dumb on your repo, the fix is usually one more rule there.

## What needs an issue first

- **Changes to the backlog format** (`docs/backlog-format.md`). The format is the public contract — breaking it breaks every port. Propose in an issue, tag it `format-v2`.
- New skills. Ratchet stays a four-skill story (audit → backlog → loop → ship); new capabilities should usually extend an existing skill.

## House style

- Skills are **single-file** (`skills/<name>/SKILL.md`), English, imperative voice, no fluff.
- Every rule in a skill must earn its place: it prevents a specific, nameable failure.
- Examples over adjectives.

## Testing a skill change

Copy the modified folder into `~/.claude/skills/`, run it against a scratch repo, and paste the transcript highlights into your PR description. There is no CI that can judge prompt quality — reviewers read transcripts.
