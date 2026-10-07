# 2026-05-26 — T.27 playtest CLI

## Context

The engine is feature-complete through T.21 / T.24 / T.26 (unified combat
engine, abilities, statuses, role-aware enemy formation, bosses, map effects,
weather cache). The UI is still the placeholder counter from the scaffold.
For two weeks every observation of combat behavior has gone through pytest —
correctness is well-covered, but feel, balance, and qualitative validation
are opaque.

This entry covers the design + first shipment of the dev-facing playtest
surface that closes that gap.

## What shipped

`tools/playtest/` — a stand-alone CLI suite that imports only from
`src/game/` (V.1 holds), with no Flet dependency:

| Script | Job |
|---|---|
| `sim_fight` | One-shot battle: pick team + enemies + weather, render tick log |
| `sim_node` | Generate any node by `(stage, node_index, run_seed)` and resolve it (FIGHT / REWARD / CHALLENGE / BOSS) |
| `sim_run` | Walk the full 50-node route with a fixed team, abort on wipe, optional CSV per-node summary |
| `inspect` | Roster browser with filters; `--show-favor WX` adds a Weather-Favor-modified column block |
| `inspect_node` | Encounter preview without combat resolution |

Plus `_common.py` for shared parsing, table formatting, `default_team`, and
the boss-combat helper (the one wiring `resolve_combat` can't do because it
doesn't accept a `map_effect_id`).

32 new tests live under `tests/tools/` (16 unit, 12 smoke, 4 helpers).
Full suite is now 563/563.

## Design discoveries

### The two-engine ghost

The original plan draft was written before T.26 landed, when the codebase
still had two combat engines side by side. Re-reading the docs against
current code surfaced that T.26 (PR [#18][pr18], commit `b229f93`) had
already unified them:

- `resolve_combat` now delegates to `compile_loadout → CombatContext →
  combat/loop_new.run → BattleResultRecorder.build_result`.
- `compile_loadout` now applies Weather Favor (the gap I had flagged in the
  earlier engine-split note).
- `combat_log.format_combat_log` renders results from the unified engine
  without modification.

The original plan called for a `--engine` toggle and a custom
`DebugRecorder`. Both got deleted from the plan during the reality check —
they would have been pure cargo-cult. The `engine_split.md` note was
demoted from "current state" to "historical context, resolved by T.26".

[pr18]: https://github.com/medukn/tempest-fauna-trail/pull/18

### Boss fights still need bespoke wiring

`resolve_combat(team, enemies, weather)` is the public single-entry contract,
but boss fights need their `map_effect_id` attached to the `CombatContext`
before the loop runs. That can't happen through `resolve_combat`'s
signature.

The playtest layer absorbs this as `resolve_boss_combat(team, encounter,
weather, *, run_seed, node_id)` inside `_common.py`. It composes the same
primitives `resolve_combat` does, plus `attach_map_effect`:

```
compile_loadout → speed_tiebreaker → assign_spawns →
recorder.register(bus) → CombatContext → attach_map_effect → run + build_result
```

`sim_node` and `sim_run` both call it for BOSS_FIGHT nodes. If a future T.X
extends `resolve_combat` to accept an optional `map_effect_id`,
`resolve_boss_combat` collapses to a one-liner.

### Qualitative signal hits immediately

First run of `sim_run --run-seed 12345 --weather-strategy stage-affinity`
with the default stub team (tier-1 trio) cleared 3 nodes and drew on
node 4. With a hand-picked tier-10 team it cleared all 9 standard nodes of
stage 1 and drew on the Holloway boss fight. Both results feel right:
stage 1 bosses are supposed to be a wall, and a stub team has no business
clearing them. The tools are surfacing the boss as a real difficulty
inflection without any extra instrumentation.

A few minutes after sim_fight worked, the same logging that powers
`format_combat_log` was visible against ability casts and Affinity Clash
multipliers — the kind of detail you can only see in a tick trace, and
which a pytest assertion would not have caught when the numbers feel
slightly off.

## Decisions that came out of the work

- **No `--engine` flag.** One engine, one default. Avoids reintroducing
  a distinction the codebase no longer makes.
- **No DebugRecorder.** `BattleResultRecorder` ships with the engine;
  rendering goes through `combat_log.format_combat_log`. Anything missing
  from the renderer (status / heal / spawn events) is a follow-up on
  `combat_log` itself, not a parallel implementation.
- **Boss wiring in `_common.py`, not duplicated.** `sim_node` and `sim_run`
  both call `resolve_boss_combat`. If the API ever changes, one place to
  update.
- **No admin Flet view yet.** Layer 2 of the plan stays deferred. CLI
  unblocks balance work on its own; a UI panel can come later (and reuse
  these same functions).

## Lingering threads

- `combat_log._format_event` only renders MOVE / ATTACK / CAST / DEATH.
  Status applies, heals, and spawns from the new ability framework fire on
  the bus but never reach the text log. Out of scope for T.27; small
  follow-up.
- `combat/legacy.py` still holds the public `resolve_combat` shim plus a
  pile of deprecated helpers re-exported through `combat/__init__.py` for
  old test compatibility. Cleanup task: move the shim out of `legacy.py`,
  delete the dead helpers, drop `combat/loop.py`. Noted in
  `engine_split.md`.
- Default stub team picks at `tier == stage_index`. Cleared all of stage 1
  with the T10 set, but stage-3+ stub teams will probably wipe early.
  T.25's matchup sweeps will give a principled answer; for now `--team`
  override is enough.

## Files touched

- `docs/design/playtesting/{README.md, plan.md, engine_split.md}` — design.
- `tools/playtest/{__init__.py, _common.py, sim_fight.py, sim_node.py,
  sim_run.py, inspect.py, inspect_node.py}` — implementation.
- `tools/__init__.py`, `tests/tools/{__init__.py, test_playtest_common.py,
  test_playtest_smoke.py}` — package + tests.
- `SPEC.md` — added T.26 + T.27 task rows, expanded V.2, updated
  Implementation Order.
- `CLAUDE.md` — updated quick reference, project structure, documentation
  map, V.2 description.

563/563 tests pass.
