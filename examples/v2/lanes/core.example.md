# LANE core — acme-todo API hardening
<!-- ratchet:v2:lane core -->

> The **default lane** (Scope `(rest)`): everything no named lane owns — dependencies and the
> lockfile, shared config, generated files, and integration fixes. It is a **barrier lane**:
> a `--lane core` run may only start when every other lane is closed (last run marker is
> `run ended`) and merged (no `ratchet/<lane>` branch ahead of the integration branch). That
> is what makes UI-06 — which adds `@tanstack/react-virtual` and touches `package.json` +
> `package-lock.json` — safe to run at all: nothing else is in flight to conflict with it.
>
> Note UI-06 lives here rather than in the `ui` lane despite editing `web/src/TodoList.tsx`:
> an item belongs to the default lane as soon as *any* path its Spec implies falls outside
> every named lane's scope. Lane assignment follows the diff, not the theme.
>
> This lane is **closed** — it has no run markers at all, because no executor has ever opened
> it. Both journal lines below are the curator's, written on the integration branch.

## Ledger
| ID | Status | Attempts | Commit | Note |
|---|---|---|---|---|
| UI-06 | todo | 0 | — | barrier item: adds a dependency (package.json + lockfile) |
| OPS-04 | needs-human | 0 | — | [OPS] requires staging server access |

## Journal
- 2026-07-07 OPS-04 filed straight to needs-human by the curator: the `[OPS]` gate needs staging
  server access, so no run should ever pick it up. (No executor has opened this lane — hence no
  run markers below. Both lines here are the curator's, written on the integration branch.)
- 2026-07-07 UI-06 filed in `core`, not `ui`: its Spec names a new dependency, so its diff
  necessarily escapes `web/src/**`. Two lanes each adding a dependency in the same wave would
  collide on the lockfile — the barrier is what prevents that.
