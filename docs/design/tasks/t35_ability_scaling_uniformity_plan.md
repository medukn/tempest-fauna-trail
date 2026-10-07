# T.35 Plan — Ability scaling uniformity + dead-stat balance (issue #42)

> **Status:** plan — ready for review. **New §T rows** (T.35a + T.35b); not in SPEC yet.
> **Depends:** T.34a/b/c (✅ Done — `ability_text.py`, `ABILITY_META`, `ScalingTerm`/`Clause`/`AbilityMeta`, 276-meta roster); T.32/T.33 (✅ — the 6-axis composer + `_DURABILITY`/`_INTENT` tables this task re-tunes). Nothing unbuilt gates it.
> **Resolves:** [#42](https://github.com/medukn/tempest-fauna-trail/issues/42) — **Finding A** (renderer blind to Tier-B inline scalers + free-prose drift) in T.35a; **Finding B** (dead-INT roster + tanks carrying too much primary stat) in T.35b.
> **Design source of truth:** issue #42 (Finding A: A1/A2/A3; Finding B: groups 1/2 + B1/B2/B3; the owner's "why-not-one-class" comment); the GAS reference pattern (`EGameplayEffectMagnitudeCalculation` = `{ScalableFloat, AttributeBased, CustomCalculationClass, SetByCaller}` — a **closed polymorphic** magnitude set, not one shape); SPEC §V.33 (intent ±10% HP·DPS proxy), §V.34 (scaling), §V.38/§V.39 (ability text); `docs/live/content/abilities.md`.
> **What this plan adds beyond those:** a concrete `Magnitude` Protocol + 4 kinds mapped to the audited residue; the A2 AST guard contract; the `Clause.terms` (A1) shape; **the verified re-tune numbers** for `_DURABILITY`/`_INTENT` (proxy-checked); the per-role dead-INT coefficient policy; the axis↔scaling §V; a determinism policy (T.35a byte-identical / T.35b deterministic-but-re-baselined, **no sim sweep** per user).
> **Not §T rows yet** — needs `/spec` to add T.35a + T.35b, add §V.46 + §V.47, extend §V.38, add §B.19, update §D + Implementation Order. Do not edit SPEC inline.

---

## 0. Substep split

Split along a **real seam: representation (no behavior change) vs balance (behavior change).** Both ship this cycle.

- **T.35a — Magnitude family (Finding A).** Promote `ScalingTerm` into a closed `Magnitude` family; A2 orphan-stat-read guard; `Clause.terms` (A1); renderer → pure per-kind dispatch. **Byte-identical** — sims unmoved (V.2/V.14). Pure representation refactor.
- **T.35b — dead-stat balance (Finding B).** Depends on T.35a (its Clause-terms machinery is how the new INT scalers stay drift-safe + rendered, and A2 makes Tier-B INT visible). Two balance moves: **(1)** re-tune `_DURABILITY` + `_INTENT` so a primary-stat tank no longer rivals an assassin's primary; **(2)** add per-role-tuned INT coefficients to every dead-INT carrier. Plus the **axis↔scaling §V guard**. **Deterministic but NOT byte-identical** — re-baselines stat-dependent snapshots; **per user, we do NOT run the win-rate sim sweep** — balance ships unvalidated-by-sim (deterministic correctness still tested).

## 1. Scope

**T.35a in scope:** `game/registries.py` (Magnitude family + Protocol; `Clause.template`/`terms`); `game/ability_text.py` (per-kind render dispatch); `game/abilities/{champions,enemies,bosses}.py` (relocate audited inline scalers into magnitudes — byte-identical); `tests/game/test_ability_text.py` (kind correctness, A2 guard, snapshot regen — text only); `docs/live/content/abilities.md`.

**T.35b in scope:** `game/content.py` (`_DURABILITY`, `_INTENT` re-tune); `game/abilities/{champions,enemies}.py` (per-role INT coeffs on ~13 dead carriers, authored as magnitudes); `tests/game/{test_role_intent,test_content,test_scaling}.py` (proxy guard re-verify, absolute-value updates, axis↔scaling guard); regen `ability_formulas.snapshot.json`; `docs/live/content/{abilities,rosters}.md`.

**Out of scope (why):** `ui/` tooltip surfaces → UI phase. `tools/export_roster.py` → passive consumer, richer output free. The **sim win-rate sweep** (`tools/simulation/`) → user opted out for this pass ("fix B without sims"); a follow-up sim-tuning pass can refine the first-pass numbers later. Squishy-durability buff → left as a deferred lever (§7) — the approved re-tune touches tanky + intent only.

## 2. The gap today

### 2a. Finding A — two number-shapes, renderer sees one (full grep audit)

| piece | where (`file.py:line`) | category | state |
|---|---|---|---|
| `ScalingTerm` (linear headline) | `registries.py:186` | Cat-0 | ✅ rendered |
| steam_engineer shred `-(8+INT*0.15)` | `enemies.py:770-771` | Cat-0 linear | 🔴 inline + prose |
| generic `INT*coeff` | `enemies.py:835` | Cat-0 linear | 🔴 inline |
| hierarch armor/res `20+INT*0.4`/`10+INT*0.2` | `enemies.py:1249-1250` | Cat-0 linear | 🔴 inline + free-prose (drift-exposed) |
| hierarch barrier `50+INT*2.0` | `enemies.py:1280` | Cat-0 linear | 🔴 inline + free-prose |
| inquisitor/reaver/jackal `max(STR,INT)*coeff` | `enemies.py:966,2124`, `champions.py:2785` | Cat-1 max_of | 🔴 inline + prose |
| 7× %-max-HP heals | `champions.py:350,377,737,765`, `enemies.py:357,1469`, `bosses.py:675` | Cat-2 pct_max_hp | 🔴 inline; **`Piece.stat("max_hp")==0`** (`effects.py:93`) |
| turret/clone statline fractions | `enemies.py:599-601`, `champions.py:2008-2017` | Cat-3 summon | 🔴 inline dict |
| iron_maiden `STR*0.5 + stacks*5` | `enemies.py:1102` | Cat-4 runtime-stacks | 🔴 inline |
| inquisitor gate `if target.INT>target.STR` | `enemies.py:965` | Cat-5 predicate (reads target) | prose |
| caracal execute `*1.5 when hp<30%` | `champions.py:392,401` | Cat-5 conditional mult | constant + free-prose |

**Root cause:** handlers do **free inline math outside any introspectable object**. Drift (handler `*0.4` vs clause `"40%"`) and empty `formula` both follow from that split; only headline terms are snapshot-pinned (V.38).

### 2b. Finding B — primary-stat tanks rival assassins; dead INT

Verified against the **real roster** (the user's example, T5L1):

| unit | axes | STR | INT |
|---|---|---|---|
| `champ_coral_colossus` (tank) | `stat=str`, tanky_hp, utility | **92** | 8 |
| `champ_duskstep_marten` (assassin) | `stat=int`, ability, damage, blinding | 17 | **127** |

A unit's **primary** = its `stat` axis × durability. The bug: `_PRIMARY_STAT["str"]=1.8` × `_DURABILITY["tanky_hp"]["strength"]=0.55` **= 0.99 ≈ a bruiser's 1.0** (`content.py:34,77`). So a str-tank's STR (92) nearly matches an int-assassin's INT (127) — **gap only 35**. The durability penalty barely offsets the primary-axis bonus → "tanks have too much STR/INT."

Dead INT (issue groups, confirmed at `content.py`):
- **Group 1** (`stat="int"` supports, flat abilities — INT scales *nothing*): `champ_geode_beetle` (`:533`), `champ_goldcrest_lark` (`:503`), `champ_coppercrest_stork` (`:553`), `champ_dusk_bat` (`:531`), `champ_will_o_fawn` (`:541`, mis-authored mage w/ support kit), `enemy_signal_drummer`, `enemy_standard_bearer`, `enemy_company_guard`.
- **Group 2** (`stat="hybrid"` tanks, STR==INT, kit uses only STR): `champ_goldhide_rhino` (`:506`), `champ_glacierback_mammoth` (`:526`), `enemy_iron_maiden`, `enemy_quarried_behemoth`, `enemy_stone_warden`.
- *(Steam_engineer / hierarch / company_captain looked dead but scale INT via Tier-B inline — they were only flagged by Finding A; **not** dead. T.35a surfaces them.)*

## 3. Architecture — T.35a (Magnitude family)

### 3.1 The `Magnitude` Protocol
Mirror GAS's `FGameplayEffectModifierMagnitude`: one **closed** kind-set, every kind introspectable, all behind one interface. Pure data, zero Flet (V.1).
```python
# registries.py
class Magnitude(Protocol):
    label: str
    def eval(self, source, target=None, caller=None) -> float: ...
    def render_formula(self, source) -> str: ...
    def render_inline(self, source) -> str: ...
```
`AbilityMeta.terms: tuple[Magnitude, ...]`; `Clause.terms: tuple[Magnitude, ...]` (new).

### 3.2 The 4 kinds (GAS analog · absorbs · byte-identical to)
| kind | GAS analog | Cat | `eval` | identical to |
|---|---|---|---|---|
| **`ScalingTerm`** *(kept)* | AttributeBased+ScalableFloat | 0 | `_eval_scaling(base,scaling,source)` | itself (sig gains unused `target`/`caller`) |
| **`PctResource`** | AttributeBased(target) | 2 | `_resolve(source,target).max_hp * pct` (`of="self"\|"target"`) | `owner.max_hp * _PCT` |
| **`MaxOfTerm`** | CustomCalc | 1 | `base + max(source.stat(s) …) * coeff` | `max(STR,INT)*_COEFF` |
| **`SetByCaller`** | SetByCaller | 4 | `base + caller[key] * coeff` | `stacks*5` |

- `PctResource` reads `.max_hp` **directly** (not `.stat()`), dodging the `Piece.stat("max_hp")==0` trap (`effects.py:93`). `of="target"` makes cross-entity %-of-target-max-HP first-class (none in roster yet — door open without another refactor).
- `SetByCaller`: handler injects `caller={"stacks":n}`; renders the **rate** ("+5 per stack"), no pre-combat total. Iron_maiden → `ScalingTerm("damage",0,"strength*0.5")` + `SetByCaller("per_stack",0,5,key="stacks")`, summed — byte-identical.

### 3.3 Renderer → pure dispatch
`render` maps over `meta.terms` + each `clause.terms`, calling the magnitude's own `render_formula`/`render_inline`/`eval`. `ScalingTerm`'s impls are today's `_format_scaling`/`_scaling_inline` bodies moved onto the class (`_STAT_SHORT`/`_short` stay shared helpers). **Deletes** the ScalingTerm-only branch (`ability_text.py:84-159`). Token substitution shared by blurb + clause templates.

### 3.4 `Clause.terms` (A1)
```python
@dataclass(frozen=True)
class Clause:
    text: str = ""                 # static prose (unchanged path)
    template: str = ""             # OR "{token}" template filled from terms
    terms: tuple[Magnitude, ...] = ()
```
Hierarch single-sourced: `HIERARCH_ARMOR = ScalingTerm("armor",20,"intelligence*0.4")`; handler reads `HIERARCH_ARMOR.eval(actor)`; meta clause `template="Grants Armor ({armor}) and Resistance ({res})."` with `terms=(HIERARCH_ARMOR, HIERARCH_RES)`. Handler math ≡ clause prose → V.38 drift dead, extended to Tier-B.

### 3.5 New primitives + fidelity (determinism, V.2/V.14)
- No RNG; every `eval` deterministic. Each kind reproduces the **exact** float the inline expr produced (literal-for-literal swap, the T.34 method). Verify sim hashes **unmoved** (champion/enemy/boss batteries). Only intended diff: `ability_formulas.snapshot.json` *text* (new kinds now render numbers) — re-baselined once.
- Cat-3 summons → `SummonSpec` (§4.4); Cat-5 predicates stay code (§4.5).

## 3'. Architecture — T.35b (balance)

### 3'.1 `_DURABILITY` re-tune (`content.py:56-89`) — **approved first-pass**
| key | strength/intelligence | old → new |
|---|---|---|
| `tanky_hp` | `0.55 → 0.42` |
| `tanky_arm` | `0.55 → 0.42` |
| `squishy` / `hybrid` | **unchanged** (squishy lever deferred, §7) |
HP/armor/res/threat on tanky **unchanged** — tanks keep durability, just hit softer. No proxy guard on durability (free to change; it's the intended balance shift).

### 3'.2 `_INTENT` re-tune (`content.py:117-139`) — **approved first-pass, proxy-verified**
```python
"damage":  {strength/intelligence 1.08→1.14, attack_speed 1.05→1.04,
            max_hp 0.96→0.93, armor 0.97→0.94, resistance 0.97→0.94,
            mana_regen 0.97 (keep), threat 0.92 (keep)}
"utility": {strength/intelligence 0.94→0.87, attack_speed 0.96→0.97,
            max_hp 1.08→1.12, armor 1.05→1.06, resistance 1.05→1.06,
            mana_regen 1.08 (keep), threat 1.10 (keep)}
```
Proxy `dmg·AS·√(hp·arm·res)` = **1.075** (damage) / **0.947** (utility) — both in `[0.90,1.10]` (V.33, verified by computation). `str==int` preserved (proxy guard line 203).

**Combined effect on the user's metric** (Coral STR vs Marten INT, T5): `92 vs 127 (gap 35)` → **`65 vs 134 (gap 69)`**. Tanky penalty bites STR harder *and* intent widens it (Coral=utility ↓, Marten=damage ↑).

### 3'.3 Per-role dead-INT coefficients (~13 carriers)
Each dead carrier's primary ability outlet gains an **INT-scaled `ScalingTerm`** authored via the §3.4 Clause-terms machinery (so it's combat-source-of-truth + rendered + A2-covered). **Per-role sizing** (user choice) — magnitude fits the outlet's role, not a flat constant. First-pass coefficients (§5). Group-2 tanks **also** get INT coeffs (not reclassified — user chose per-role tuned over the reclassify split).

### 3'.4 Axis↔scaling guard (§V.47, B3)
Test: a `ChampionDef`/`EnemyDef` with `stat="int"` **must** reference INT via some `Magnitude` on its active/passive meta; `stat="hybrid"` must reference **both** STR and INT; `stat="str"` must reference STR. Relies on A2/T.35a making every scaler a visible `Magnitude`. Stops dead-stat recurrence (CI-guarded, mirrors V.22/V.38).

## 4. Decisions

- **4.1 Keep `ScalingTerm` name** for the linear kind — 276 metas + every handler use it; the diff is a pure *addition* of 3 sibling kinds + Protocol, not a rename. Lowest byte-identical risk.
- **4.2 `eval(source, target=None, caller=None)`** — superset of `eval(source)`; existing calls identical.
- **4.3 A2 guard = coarse + allowlist.** AST-walk each registered handler (`inspect.getsource`), collect every `X.stat("<lit>")` / `.max_hp` / `.hp` read; assert covered by an `AbilityMeta` magnitude (terms or any clause-terms; `PctResource`⇒max_hp, `MaxOfTerm`⇒its stats) **or** on `_PROSE_ALLOWLIST: dict[id,reason]`. Conservative on purpose (AST can't cheaply prove source-vs-target-vs-predicate); a false positive costs one allowlist line, never a silent gap. Seeds: Cat-5 predicates, summons (if inline), flat `max_hp +=` growth.
- **4.4 Cat-3 summons → `SummonSpec`** (recommended; only 2 sites): a frozen dataclass holding the summon `base_stats` template where fractions are `Magnitude`s (reuses the family). Handler builds `Piece` from `spec.eval(owner)`; renderer describes it. **Fallback:** keep inline on the allowlist. The one defer-able sub-piece.
- **4.5 Cat-5 conditional:** predicate (`hp_pct<0.3`, `target.INT>target.STR`) stays handler code (allowlisted); the *numbers* (`1.5`, `30%`) move to clause-terms so `+50%` can't drift from `1.5`. Threshold `0.3` is a bound, not an outlet (A3 honesty).
- **4.6 `will_o_fawn`** (mis-authored mage w/ support kit): per-role tuning gives its utility outputs INT scaling (e.g. the AS-buff magnitude / INT grant scales with caster INT). The mage-with-no-damage *role* mismatch is flagged as a known content wart for a later pass — **not** reclassified now (overridable, §7).

## 5. Authored values

**T.35a:** none — relocates existing constants into magnitudes (representation only).

**T.35b re-tune:** §3'.1 / §3'.2 tables (proxy-verified). **First-pass / tunable.**

**T.35b dead-INT coefficients (first-pass, per-role; tunable):**
| outlet type | example carriers | coeff |
|---|---|---|
| big single-target / team shield (armor/res) | geode_beetle (+80 armor/+40 res), coppercrest_stork (+50 armor) | `+0.35×INT` |
| flat stat team-buff (STR/AS) | goldcrest_lark (+20 STR), standard_bearer (+12 STR), signal_drummer (+15 AS), dusk_bat (−30 AS) | `+0.15×INT` |
| group-2 tank ability outlet (STR-based dmg/effect) | goldhide_rhino, glacierback_mammoth, iron_maiden, quarried_behemoth, stone_warden | `+0.20×INT` added component |
| will_o_fawn utility outputs | will_o_fawn | `+0.15×INT` |
Start small per your ask; a later sim pass refines.

## 6. Content / roster audit + reconciliation

- **§B.19 (new) — Finding A drift class:** Tier-B inline scalers invisible in `formula` + free-prose clauses can diverge from handler math (hierarch `*0.4` vs `"40%"`). Origin: T.34 scoped headline-only into terms. Fix = Magnitude family; guard = §V.46 (A2).
- **§B.20 (new) — Finding B structural:** `1.8 (primary axis) × 0.55 (tanky) ≈ 0.99` let a primary-stat tank rival an assassin's primary (Coral STR 92 vs Marten INT 127). Origin: durability penalty sized without accounting for the primary-axis 1.8 multiplier (T.32/T.33). Fix = `_DURABILITY` re-tune; guard = the proxy band stays (V.33) + the new axis↔scaling guard catches dead stats.
- **Living-doc drift:** `docs/live/content/abilities.md` Tier-B paragraph ("max_hp/max_of stay inline + clause / can't be a ScalingTerm") becomes false after T.35a → rewrite to the Magnitude taxonomy. `rosters.md` stat-profile notes update after the re-tune.
- 276-meta coverage (V.38) preserved exactly; no id added/removed.

## 7. Open questions

**Resolved here (overridable):**
- Full GAS-style closed set (4 kinds) + A2 + Clause-terms — *user-chosen*.
- Keep `ScalingTerm` name (4.1); `SummonSpec` built not allowlisted (4.4); coarse A2 guard (4.3).
- Re-tune = **first-pass** (tanky 0.42, intent 1.14/0.87) — *user-chosen*.
- Dead-INT coeffs = **per-role tuned** (§5) — *user-chosen*; group-2 tanks get coeffs (not reclassified).
- `will_o_fawn` gets INT-scaled utility, role mismatch flagged not fixed (4.6).

**Still open / deferred:**
- **No sim sweep this pass** (user: "fix B without sims") — the re-tune + coeff numbers ship deterministic-but-sim-unvalidated; a later `tools/simulation/` pass can refine first-pass values.
- **Squishy-durability buff** (1.25→1.35) as a further offense lever — deferred; approved re-tune touches tanky + intent only.
- UI consuming per-kind structured data vs the flat `formula` string — UI phase.

## 8. Test plan

**T.35a (byte-identical):**
- V.38 coverage unchanged green (all 276 render).
- **A2 guard** `test_no_orphan_stat_reads` — covered-or-allowlisted; inject an uncovered read in a fixture → assert it **fails** (guards itself).
- Per-kind correctness: `PctResource(0.03).eval(goldhide)==goldhide.max_hp*0.03`; `MaxOfTerm.eval(reaver)==max(STR,INT)*COEFF`; `SetByCaller.eval(im,caller={"stacks":4})==20`.
- **Determinism (load-bearing):** champion+enemy+boss sim batteries (T.34 harness, fixed seed, `workers=1`) → result hashes **unmoved** pre/post. The whole point of the a/b split.
- Render smoke (276 ids, no leftover `{token}`); clause-templates fully substituted.
- Snapshot regen `ability_formulas.snapshot.json` — diff is only intended Tier-B formula gains.
- Purity (V.1) extended to new kinds.

**T.35b (deterministic, re-baselined, no sim sweep):**
- **Proxy guard** `test_hp_dps_proxy_within_10pct` still passes with new `_INTENT` (computed: 1.075 / 0.947).
- **Axis↔scaling guard** `test_axis_scaling_alignment` (§V.47): every `stat=int`/`hybrid`/`str` def references the matching stat via a Magnitude; inject a dead-INT def → fails.
- Update absolute-value assertions in `test_content`/`test_scaling` to the re-tuned numbers (mechanical).
- Determinism (V.2): same seed → identical output post-re-tune (deterministic, just different from pre-re-tune). Regen `ability_formulas.snapshot.json` (stat values changed → rendered numbers changed).
- **Explicitly NOT run:** the `tools/simulation/` win-rate sweep — per user. Note in the journal that balance is sim-unvalidated this pass.

## 9. Acceptance criteria

**T.35a:**
1. `Magnitude` Protocol + `ScalingTerm`(unchanged)/`PctResource`/`MaxOfTerm`/`SetByCaller`; `Clause` gains `template`+`terms`.
2. Every Cat-0/1/2/4 inline scaler (§2a) reads via a magnitude; drift-prone clause numbers live in clause-terms.
3. `render` dispatches over magnitude hooks; ScalingTerm-only branch gone; 276 ids render clean.
4. A2 guard passes with documented `_PROSE_ALLOWLIST`; injected orphan fails it.
5. Sim hashes **byte-identical** across all batteries (V.2/V.14).
6. `ability_formulas.snapshot.json` regen; diff only intended Tier-B gains.
7. Summons via `SummonSpec` (or allowlisted w/ reason); Cat-5 predicates allowlisted.
8. `docs/live/content/abilities.md` updated; `/check` clean.

**T.35b:**
9. `_DURABILITY` tanky STR/INT = 0.42; `_INTENT` damage 1.14 / utility 0.87 (full tables §3'); proxy guard green.
10. Coral STR ≈ 65, Marten INT ≈ 134 at T5 (gap ≈ 69) — verified by a content test.
11. All ~13 dead-INT carriers reference INT via a Magnitude (§5 coeffs); axis↔scaling guard (§V.47) green; injected dead-INT def fails it.
12. Absolute-value tests + `ability_formulas.snapshot.json` re-baselined; full suite green **except** the (un-run) sim sweep.
13. `docs/live/content/{abilities,rosters}.md` updated; `/check` clean.

## 10. SPEC changes needed (for `/spec`)

- **§T — add rows:**
  - `T.35a | Ability scaling uniformity — closed `Magnitude` family (`ScalingTerm` linear + `PctResource`/`MaxOfTerm`/`SetByCaller`, GAS-modeled) behind one `eval(source,target,caller)` Protocol; `Clause.terms`+template (A1); `ability_text` → pure per-kind dispatch; A2 AST orphan-stat-read guard + allowlist; all Tier-B inline scalers relocated into magnitudes (byte-identical, V.2/V.14); snapshot regen | `game/registries.py`, `game/ability_text.py`, `game/abilities/{champions,enemies,bosses}.py`, `tests/game/test_ability_text.py`, `docs/design/tasks/t35_ability_scaling_uniformity_plan.md` | T.34a/b/c | M | 📋 Plan`
  - `T.35b | Dead-stat balance (#42 Finding B) — re-tune `_DURABILITY` tanky STR/INT 0.55→0.42 + `_INTENT` damage 1.14/utility 0.87 (proxy-verified, V.33) so a primary-stat tank no longer rivals an assassin's primary; per-role INT coeffs on ~13 dead-INT carriers (authored as magnitudes via T.35a); axis↔scaling §V guard. Deterministic re-baseline, NO sim sweep (sim-unvalidated by choice) | `game/content.py`, `game/abilities/{champions,enemies}.py`, `tests/game/{test_role_intent,test_content,test_scaling,test_ability_text}.py` | T.35a | M | 📋 Plan`
- **§V — new V.46** (Finding-A recurrence): *handlers MUST NOT read `.stat()`/`.max_hp`/`.hp` for a numeric outlet outside a registered `Magnitude`; every such read is magnitude-covered (terms or clause-terms) or on `_PROSE_ALLOWLIST` (id→reason); CI-guarded (`test_no_orphan_stat_reads`); the `Magnitude` family (`ScalingTerm`/`PctResource`/`MaxOfTerm`/`SetByCaller`) is closed, GAS-modeled, pure, RNG-free (V.2/V.14), self-describing.*
- **§V — new V.47** (Finding-B recurrence): *every `ChampionDef`/`EnemyDef` `stat="int"` references INT via a `Magnitude` on its ability meta; `stat="hybrid"` references both STR and INT; `stat="str"` references STR — CI-guarded (`test_axis_scaling_alignment`), mirrors V.22/V.38. (Auto reads `1.0×STR+0.2×INT` count for STR only.)*
- **§V.38 — extend:** source-of-truth B covers **all** magnitude kinds + clause-terms (not only headline ScalingTerms).
- **§V.33 — note:** `_INTENT` re-tuned (damage 1.14 / utility 0.87); proxy band `[0.90,1.10]` **unchanged and still holds** (1.075 / 0.947).
- **§B — add B.19** (Finding A drift class) + **B.20** (Finding B: `1.8×0.55≈0.99` tank-rivals-assassin; fix = durability re-tune + V.47).
- **§D — update:** record dead-stat balance done (T.35b) with the caveat **sim-unvalidated this pass**; squishy lever + sim refinement noted as open.
- **Implementation Order:** T.35a in the tooltip/polish lane (post-T.34); T.35b after T.35a.

## 11. LIVING docs to update

- `docs/live/content/abilities.md` — replace Tier-B caveat with the `Magnitude` taxonomy (4 kinds, A2 guard, `Clause.terms`). Same commit as T.35a build. `/check`.
- `docs/live/content/rosters.md` — durability/intent stat-profile notes + dead-INT-now-scaling, after T.35b. `/check`.
- `docs/live/systems/effects.md` — sync term/clause shapes to the Protocol if referenced (verify at build).

---

### Next moves (in order)
1. **`/spec`** — apply §10 (add T.35a + T.35b rows, V.46 + V.47, extend V.38, note V.33, add B.19 + B.20, update §D + Implementation Order).
2. **`/build §T.35a`** — byte-identical refactor (sim-hash test is the gate).
3. **`/build §T.35b`** — balance re-tune + dead-INT coeffs + axis↔scaling guard; re-baseline snapshots; **no sim sweep**.
