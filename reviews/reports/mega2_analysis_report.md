# Mega2 Simulation Analysis Report

*Deep analysis of the `results/mega2/` sweep. Backing document for [issue #28](https://github.com/medukn/tempest-fauna-trail/issues/28).*

Generated 2026-05-30 from `results/mega2/` (1v1 full round-robin ×6 weathers, 2v2 sampled ×6 weathers, 3v3 sampled on clear).

---

## ⚠️ Rectification (added 2026-05-30, post-publication)

**This dataset was generated on a build where no designed abilities fire.** Discovered while drafting per-piece fixes. Two compounding facts:

1. **Ability content is unimplemented** (SPEC §D.5: the T.20 *framework* is done, per-piece *kits* are still open). Only ~16 of 240 roster ability slots have any coded handler.
2. **Those ~16 handlers are mis-registered** — coded under short ids (`torrent_heron.active`) while the roster references prefixed ids (`champ_torrent_heron.active`). **0 of 240 roster ability-ids resolve to a handler** (empirically verified).

So every piece falls back to the generic cast in [loop_new.py:334](../src/game/combat/loop_new.py#L334): `raw = 0.2*strength + 4.2*intelligence` (INT coeff 21× STR), plus auto-attacks. Combat in this dataset = stat-sticks + auto-attacks + one identical int-scaled nuke.

**What survives this caveat** (structural, independent of kits): §1 the win-curve cliff, §2.2 `ΣP` distribution-blindness, §8.1 champion edge, aggregate balance. These are about HP/DPS/focus-fire and hold regardless.

**What is invalidated / provisional** (anything kit-dependent):
- §3 role balance — "str/warrior weak, int/mage fine" is largely an artifact: the generic nuke scales on INT, so int pieces get a free strong "ability" and str pieces get ~84 dmg vs ~756 (see §7). Real kits will redistribute this.
- §4 trait (Kinship/Calling) deltas — measured on stat-sticks, not kits.
- §6–7 most piece outliers, especially the str+ability pieces (Powder Sapper et al.): they are **crippled by the int-scaled fallback, not mis-designed**. Their designed kits scale on STR and have never fired.
- Tier drift §2.3 is partly real (cliff) but its role/kit component is unmeasured.

**Action:** implement the kit catalog + fix the registration bug, then **re-run mega**. Only then are per-piece/role/trait verdicts in §3–7 trustworthy. Tracking: [`docs/design/tasks/t30_ability_catalog_plan.md`](../docs/design/tasks/t30_ability_catalog_plan.md).

---

## 0. Executive summary

The roster's **aggregate balance is healthy** (champion vs enemy win rate is centered in every format) and the **base scaling math is sound** (`P`, `√P` coupling). The problems are *structural* and *within-roster*, and they trace back to two engine-level facts:

1. **Combat resolves almost deterministically.** Win rate is a near-step function of the power ratio (Section 1). The contested band is narrow (~0.82×–1.5× power), fights are decisive even at parity, and the shape is identical across 1v1/2v2/3v3.
2. **`ΣP` does not capture team strength.** At equal total power, a tier-balanced team beats a top-heavy one by up to ~35 pp (Section 2.2). Power distribution matters as much as power total.

Everything else — tier drift, the apparent role/trait imbalances, the brittleness of difficulty knobs — is largely downstream of these two facts. Specific actionable items: a hard-broken piece (Powder Sapper, 0% solo), a chronically weak warrior role, a catastrophic low-tier floor, and a genuinely strong **Hunter** calling that survives tier-controls.

---

## 1. Dataset and method

| Stage | Weathers | Battles each | Min matches/piece | Timeouts |
|---|---|---|---|---|
| 1v1 (round-robin) | 6 | 7,140 | 119 | 0 |
| team2-sample (2v2) | 6 | 33,000 | 1,010 | 0 |
| team3-sample (3v3) | clear only | 20,000 | 916 | 0 |

- All runs used `--max-ticks 1_000_000`, disabling sudden death → `timeout_rate = 0` everywhere (Section 8.3 caveat).
- 120 pieces total (60 champions, 60 enemies).
- **Metrics:** `win_rate`; `expected_wr` (deterministic power-threshold model from `power(T,L)`); `wr_delta = win_rate − expected_wr`; `mean_duration_ticks`.
- **Win-curve / power analysis:** team `ΣP` computed via `power(tier, 1)` ([src/game/scaling.py](../src/game/scaling.py)) as a level-1 proxy — per-battle level isn't in `results_*.csv`. Level noise adds scatter but does not change monotone shapes.

---

## 2. Combat structure and power scaling

### 2.1 The win-curve is a cliff (team-size invariant)

Team-A win rate, binned by `Pa / (Pa + Pb)`:

| Pa share | power ratio | 1v1 | 2v2 | 3v3 |
|---|---|---|---|---|
| 0.40 | 0.67× | 0.080 | 0.134 | 0.090 |
| 0.45 | 0.82× | 0.326 | 0.307 | 0.271 |
| **0.50** | **1.00×** | **0.493** | **0.514** | **0.506** |
| 0.55 | 1.22× | 0.745 | 0.732 | 0.726 |
| 0.60 | 1.50× | 0.894 | 0.894 | 0.890 |
| 0.65 | 1.86× | 0.985 | 0.971 | 0.975 |
| 0.70 | 2.33× | 0.993 | 0.997 | 0.996 |

The three formats are nearly superimposable → this is a property of the combat engine, not of team size. The contested band is only ~`0.45–0.60` share (power ratio 0.82×–1.5×). A 1.22× edge already wins ~73%; 1.5× wins ~89%.

**Cause:** clean-duel dynamics — the higher combat value `V = HP × DPS` wins almost always (Lanchester square law). `√P` stat coupling correctly makes the stat *product* linear in `P`, but with almost no variance the *outcome* is a step function of the ratio.

### 2.2 `ΣP` is distribution-blind (the most important scaling result)

Restricting to near-parity battles (`|Pa−Pb|/avg < 10%`) and bucketing by intra-team tier-spread difference (`max_tier − min_tier`):

**3v3 (cleanest signal):**

| Δ tier-spread (A−B) | n | WR_a |
|---|---|---|
| −4 (A flatter) | 333 | 0.640 |
| −3 | 225 | 0.653 |
| −2 | 331 | 0.598 |
| −1 | 337 | 0.558 |
| 0 | 441 | 0.485 |
| +1 | 388 | 0.503 |
| +2 | 316 | 0.462 |
| +3 | 231 | 0.381 |
| +4 (A top-heavy) | 341 | 0.293 |

Monotonic: at equal `ΣP`, the **flatter-tier team beats the top-heavy team by ~35 pp** at the extreme. Direct concentration cut: the more-concentrated team wins only **0.388**. The 2v2 trend is the same direction but noisier (less room to spread with 2 pieces): Δ−4 → 0.635, Δ+4 → 0.361.

**Cause:** action economy + focus fire. A `T9 + T1 + T1` squad spends its whole budget on one carry plus two bodies that die instantly and contribute nothing; three `T5`s all fight. Same `ΣP`, very different real strength.

**Consequence:** `ΣP` is not a valid encounter budget on its own — two squads with identical `ΣP` can differ ~35 pp. Encounter generation that buys a high-tier carry + chaff under-delivers; players should spread tiers, and over-leveling a single carry is a trap (ties to T22 economy/leveling).

### 2.3 Tier drift (a metric artifact of the cliff)

Champion `wr_delta` by tier:

| tier | 1v1 | 2v2 | 3v3 |
|---|---|---|---|
| T1 | −0.155 | −0.118 | −0.109 |
| T2 | −0.116 | −0.114 | −0.116 |
| T3 | −0.259 | −0.143 | −0.127 |
| T4 | +0.020 | −0.026 | −0.018 |
| T5 | −0.078 | −0.057 | −0.051 |
| T6 | +0.093 | +0.050 | +0.025 |
| T7 | +0.045 | +0.021 | +0.045 |
| T8 | +0.159 | +0.138 | +0.103 |
| T9 | +0.226 | +0.183 | +0.153 |
| T10 | +0.163 | +0.138 | +0.118 |

Low tiers underperform their budget, high tiers overperform — the deterministic engine always forces a winner, and secondary factors compound with power differences. This is expected behaviour for the power-threshold model. It compresses as team size grows (averaging) but persists. Practical consequence: **cross-tier `wr_delta` cannot be read as kit quality** — a `--tier-stratified` sweep is required for that.

### 2.4 Fights are decisive even at parity

Winner's leftover HP (3v3): p10 462 / p25 938 / median 1598 / p75 2301 / p90 2952. In near-parity battles: p25 585 / median 989. Even the closest quartile of *fair* fights leaves the winner with real HP — nailbiters are rare. This is Section 2.1 seen at the team level: once a side gains the edge, focus fire snowballs it. Drama (comebacks, tension) must be engineered in; the base engine does not produce it.

---

## 3. Role balance

Mean `wr_delta` by role:

| role | 1v1 | 2v2 | 3v3 | read |
|---|---|---|---|---|
| hybrid | +0.104 | +0.098 | +0.082 | strong (T10 legendaries) |
| bruiser | +0.047 | +0.059 | +0.062 | role-appropriate anchor |
| assassin | +0.072 | +0.033 | +0.033 | fine |
| mage | −0.008 | −0.026 | −0.016 | fine |
| marksman | **−0.101** | −0.041 | −0.032 | low-tier floor problem only |
| warrior | −0.056 | −0.045 | −0.050 | **chronic underperformer** |

Role × tier band (2v2 champions):

| role | low (T1-3) | mid (T4-6) | high (T7-10) |
|---|---|---|---|
| assassin | −0.081 | +0.036 | +0.112 |
| bruiser | – | −0.087 | +0.021 |
| hybrid | – | – | +0.100 |
| mage | −0.100 | +0.017 | +0.202 |
| marksman | **−0.192** | – | +0.196 |
| warrior | −0.132 | −0.063 | +0.124 |

Key reads:
- **Marksman recovers with a frontline** (−0.101 solo → −0.04 in teams). Their 1v1 weakness was an artifact of dying before acting; the *low-tier* floor (−0.192) is the real issue.
- **Warrior is the one chronic underperformer** at every team size and tier band below high — survives but can't close.
- **Hybrid** is the intended-strong legendary role, marginally over budget.

### 3.1 Fight duration by role

`mean_duration_ticks` (team2 clear): marksman 7103, mage 7133, hybrid 7717, assassin 7741, warrior 7915, **bruiser 9699**. Bruisers drag fights ~35% longer than ranged — tanky, slow to resolve. This is the mechanical face of "tanks stall": high effective HP, low DPS → long fights they often still don't win (warrior). Relevant if/when sudden-death `MAX_TICKS` is re-enabled (Section 8.3).

---

## 4. Trait system: Kinship & Calling

Champions carry one **Kinship** (creature family) and one **Calling** (combat archetype). Mean `wr_delta` (team2, 6-weather avg):

**Kinship** (mild spread, ~17 pp):

| kinship | n | delta |
|---|---|---|
| Scaled | 7 | −0.095 |
| Tidekin | 6 | −0.064 |
| Swarm | 5 | −0.011 |
| Beast | 18 | +0.004 |
| Spirit | 15 | +0.054 |
| Skyborn | 9 | +0.071 |

**Calling** (large spread, ~32 pp):

| calling | n | delta | mean tier |
|---|---|---|---|
| Mystic | 6 | −0.121 | 3.3 |
| Mender | 3 | −0.102 | 1.0 |
| Guardian | 9 | −0.073 | 4.3 |
| Trickster | 3 | −0.053 | 2.7 |
| Warden | 6 | −0.010 | 4.0 |
| Skirmisher | 7 | −0.006 | 4.6 |
| Stalker | 7 | +0.015 | 6.6 |
| Bruiser | 7 | +0.036 | 6.4 |
| Primordial | 6 | +0.138 | 10.0 |
| Hunter | 6 | +0.199 | 8.8 |

### 4.1 Confound check — most of the Calling spread is tier

The Calling delta correlates tightly with the calling's mean tier (Mender/Mystic are low-tier callings, Primordial/Hunter are high-tier). So most of the spread is **Section 2.3 tier drift reappearing through trait labels**, not a trait-power problem per se.

**But controlling for tier (T7–10 champions only):**

| calling (T7-10) | n | delta |
|---|---|---|
| Bruiser | 4 | +0.066 |
| Stalker | 5 | +0.069 |
| Skirmisher | 2 | +0.080 |
| Guardian | 1 | +0.088 |
| Primordial | 6 | +0.138 |
| Hunter | 6 | +0.199 |

**Hunter (+0.199) and Primordial (+0.138) stay well above the other high-tier callings (~+0.07–0.09).** That residual is real: even among equally-high-tier pieces, the Hunter calling overperforms by ~+0.11–0.13. Primordial = the legendary hybrids (mean tier 10), Hunter = the high-tier DPS calling. **Hunter is the one trait worth a direct look** beyond the tier story. Kinship effects are too small and too tier-confounded to act on yet.

---

## 5. Weather and affinity

Favored-affinity (affinity == active weather) win rate vs everything else, 2v2:

| weather | favored WR | other WR | gap |
|---|---|---|---|
| cloudy | 0.551 | 0.493 | +0.058 |
| rain | 0.540 | 0.494 | +0.046 |
| thunder | 0.538 | 0.495 | +0.044 |
| mist | 0.530 | 0.496 | +0.034 |
| snow | 0.499 | 0.501 | −0.002 |
| clear | 0.468 | 0.517 | −0.049 |

Favored affinity is worth only ~+4 to +6 pp, and snow ≈ 0. **Weather is currently flavor, not a strategic axis.** (Clear reads negative only because 40 baseline-clear enemies overpopulate it, dragging the "favored" average down.) Strengthening the triangle (~+5pp → ~+12–15pp) would make weather matter *and* add the kind of variance that softens the Section 2.1 cliff.

**Mono-affinity stacking** (3v3, near parity): a stacked-affinity team wins ~0.561 **even on clear weather**, where no triangle bonus applies (small n=237). Hints at affinity-keyed synergy in traits/passives — worth a dedicated check.

---

## 6. Team composition patterns (near-parity, 3v3)

- **Tier balance dominates** (Section 2.2): ~35 pp swing.
- **Role diversity is roughly neutral** (~0.518, n=1520). Bringing one of each role barely matters; *how you spend power across tiers* matters far more than *which roles* you bring.
- **Frontline count is marginal/slightly negative** (more-frontline team wins ~0.476, n=1980) — consistent with the weak warrior role; stacking frontline doesn't buy wins.
- **Mono-affinity** slight edge (Section 5).

Net: the dominant composition lever the sim can see is *power distribution*, not role/affinity shape. The latter are second-order (and weak) at present.

---

## 7. Outliers and one hard bug

**Top overperformers (2v2, 6-weather):** Grand Marshal (T10 warrior) +0.312 / WR 0.946; Stone Warden (T10 bruiser) +0.277; Quarried Behemoth (T9 bruiser) +0.236; Spectral Heron (T9 mage) +0.216; Cliffeyrie Eagle (T9 marksman) +0.203. Note that high-tier *tanks* top the *team* charts (they anchor a comp) whereas casters topped 1v1 — formation matters.

**Top underperformers (2v2):** Powder Sapper (T2 marksman) −0.211 / WR 0.199; Permafrost Walrus (T3) −0.203; Avalanche Engine (T5 marksman) −0.193; Torrent Heron (T3) −0.193; Boulderhide Skink (T3) −0.181; Pikeman (T2 warrior) −0.175; Conscript (T1 warrior) −0.164.

**Hard bug — Powder Sapper:** WR **0.000** in 1v1, ~0.20 in teams (beta 1.0, the floor of the dataset). Not a stat-zero bug — a pure glass cannon that dies before acting solo. Fix: baseline survivability/kit value, or guarantee encounter generation never fields it un-peeled.

**Off-trend mid-tier outlier — Avalanche Engine (T5 marksman, −0.193):** sits far below its tier band (mid-tier marksman shouldn't be this low). Likely an under-tuned kit independent of the low-tier floor; inspect individually.

---

## 8. Metric and methodology notes

### 8.1 Champion intrinsic edge
Per-piece champion vs enemy WR is consistently champion-favored: 1v1 0.511/0.489, 2v2 0.508/0.493, 3v3 0.503/0.498 (~+1.5 pp). Small per fight, compounds over a full run → "parity" encounter budgets are secretly player-favored. Encounter budget should target an *intended per-fight win rate* (set by run length and death rules), not raw power parity, and be validated with full-run `sim_run`.

### 8.2 Side/initiative bias — checked, not robust
Raw team-A win rate runs slightly >0.5 in team stages (2v2 ~0.528). But at strict near-parity (±5%) it is inconsistent across formats (1v1 0.478, 2v2 0.520, 3v3 0.486), so there is **no robust positional/initiative advantage**; the raw >0.5 is most likely a sampler asymmetry (team-A drawn marginally stronger). Documented so it isn't mistaken for an engine bug later.

### 8.3 Timeouts are blind
`--max-ticks 1_000_000` means sudden death never fired; `timeout_rate = 0` everywhere. Stall behavior (e.g. bruiser/warrior fights that can't close — Section 3.1) is invisible. Re-run at engine-default `MAX_TICKS` to surface `TO%`.

### 8.4 Historical note: Bradley-Terry removed
The original analysis included Bradley-Terry β values which spanned 176× across tiers. This was a modelling error — the engine is deterministic, not probabilistic, so BT's logistic assumption was incorrect. The expected win-rate model now uses a deterministic power threshold (step function). Historical β values in this report are superseded; prefer `wr_delta` for balance reads.

### 8.5 What is and isn't measurable here
- **Measurable & trustworthy:** aggregate balance, role/tier *direction*, the win-curve, `ΣP` distribution effect, gross outliers, the Powder Sapper bug.
- **Contaminated (needs `--tier-stratified`):** any cross-tier `wr_delta` read as kit quality (Sections 2.3, 4).
- **Missing:** team3 for 5 of 6 weathers; timeout/stall behavior; per-battle level data; economy/run-level dynamics.

---

## 9. Recommendations (priority order)

1. **Soften the win-curve with controlled variance** (crit, damage roll, stronger affinity triangle). Decide the intended `WR vs power-ratio` shape on purpose, then tune to it. Highest leverage — fixes fight feel *and* makes every difficulty/balance knob behave (Sections 2.1, 2.4, 5).
2. **Make the power budget distribution-aware** — add a concentration term to `ΣP`, or have encounter generation spread tiers; surface "don't over-level one carry" to players (Section 2.2).
3. **Replace budget-multiplier difficulty knobs** (incl. DC's `ΣP × 1.1`) with flat stat scaling + authored milestone changes — the cliff makes multipliers non-linear (Section 2.1, [issue #28](https://github.com/medukn/tempest-fauna-trail/issues/28)).
4. **Re-run with `--tier-stratified`** to recover true within-tier kit quality (Sections 2.3, 4).
5. **Fix the broken bin, not the gradient.** Powder Sapper (auto-lose), low-tier marksman/warrior floor, warrior closing power, Avalanche Engine kit. Leave the *interesting* gradient (strong legendaries, Hunter calling, high-tier prizes) — gate it, don't flatten it.
6. **Look at the Hunter calling** specifically (the one trait that overperforms tier-controlled), and the mono-affinity synergy hint.
7. **Strengthen the affinity triangle** so weather is strategic (Section 5).
8. **Re-run at engine-default `MAX_TICKS`** and complete the sweep (3v3 × all 6 weathers) to surface stalls and weather effects in 3v3 (Section 8.3).
