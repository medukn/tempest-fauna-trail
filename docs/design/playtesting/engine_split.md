# Engine Unification — Historical Note

> **Resolved.** This note is kept for context. The engine split described
> below was closed by T.26 (commit `b229f93`, PR [#18][pr18], issue [#17][i17]).

[pr18]: https://github.com/medukn/tempest-fauna-trail/pull/18
[i17]: https://github.com/medukn/tempest-fauna-trail/issues/17

## What used to be split

Before T.26 the codebase carried **two combat engines** side by side:

- **Legacy** (`combat/legacy.py`, T.3) — own tick loop, returned `BattleResult`
  with `events`. Applied Weather Favor + Affinity Clash. Ignored abilities,
  passives, statuses, board cells, map effects.
- **New** (`combat/loop.py` + `CombatContext`, T.20) — ran the full ability /
  passive / status / map-effect framework. Returned only `"team" | "enemy" |
  "draw"`. No event stream. Did not apply Weather Favor.

The T.20 plan §8.3 specified that `resolve_combat` would internally delegate
to the new loop, but the delegation was never written. Both engines stayed
live, each with its own test surface.

## How T.26 closed it

| Change | File |
|---|---|
| Added `_apply_weather_to_piece` inside `compile_loadout` so Weather Favor runs on every combat. | `src/game/loadout.py` |
| New unified tick loop with meters / pathing / status processing / map effects. | `src/game/combat/loop_new.py` |
| Bus-subscriber recorder that reconstructs `BattleResult` from the unified loop. | `src/game/combat/recorder.py` |
| `resolve_combat` rewritten as a 20-line shim: `compile_loadout → CombatContext → attach recorder → loop_new.run → recorder.build_result`. | `src/game/combat/legacy.py` |
| `combat/__init__.py` exports `run` from `loop_new`; old `loop.py` later deleted (see below). | `src/game/combat/__init__.py` |

All 388 pre-T.26 tests pass against the unified engine without modification.

## What the playtest plan inherits

Because of T.26, the playtest tooling has **one** combat entry point. The
plan's original draft assumed two and proposed:

- A `--engine legacy|new` CLI flag — **removed**, single engine now.
- A custom `DebugRecorder` for the new engine — **removed**, `BattleResultRecorder`
  ships with the engine and `combat_log.format_combat_log` renders its output.

Boss fights still need a slightly different code path because `resolve_combat`
doesn't accept a `map_effect_id`. The playtest layer composes the same
primitives manually for boss nodes — `compile_loadout → CombatContext →
attach_map_effect → loop_new.run(ctx, recorder)` — and that path is documented
inline in `sim_node.py`.

## Lingering tech debt

- `combat/legacy.py` still hosts the public shim plus deprecated helpers
  (`_apply_hit`, `_resolve_movement`, etc.) re-exported through
  `combat/__init__.py` for backward compatibility with old tests. Future
  cleanup task: move the shim out of `legacy.py`, delete the deprecated
  helpers, drop the re-exports. Not blocking the playtest work.
- ~~`combat/loop.py` (the pre-T.26 partial loop) still exists in tree alongside
  `loop_new.py`.~~ **Deleted.** It was dead production code — only
  `tests/game/test_abilities.py` still imported `run` / `process_statuses` /
  `process_casts` / `expire_modifiers` from it. That import was repointed to
  `loop_new` (all funcs present, identical signatures, 38 tests pass) and the
  file removed. `loop_new.py` is now the sole tick loop. The barrier system
  surfaced the dupe: a per-tick prune had to be written twice until the second
  copy was deleted.
