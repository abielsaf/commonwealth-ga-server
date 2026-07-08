# Recon Sniper Damage Review (Ballista / Scorpia / Dweller)

Branch: `review-recon-sniper-dmg`. Investigating player reports that recon sniper
damage is "lower than the original game." Suspects raised: player protections,
recon skills not applying, weapon mods (DDD) not applying.

Data source: `out/server.db` (the populated asset dataset — repo-root `server.db`
and `data/server.db` are 0-byte placeholders). Binary traces via Ghidra at base
`0x10900000`.

## TL;DR

- The **offense pipeline is faithful** — buff stacking, weapon mods, Super
  Sharpshooter, and Killer Instinct's enemy-debuff all verified against the binary.
- **No over-reversal bug** (earlier hypothesis DISPROVEN by log 352/354). Shooter
  output is identical shot-to-shot; damage differences are entirely target-side
  protection state.
- **KI boosts its own triggering shot — and that is RETAIL-CORRECT.** Killer Instinct
  (type 505 "Hit Situational") is submitted *before* the 264 damage in `ApplyHit`, so
  its −10 protection lands on the target before the same bullet's mitigation is
  computed. This *looks* like a bug against the tooltip ("lower his protections… for
  3 seconds"), but **clean original-server footage proves the original server did the
  exact same thing** — see [§Retail footage settles it](#retail-footage-settles-it-ki-on-first-shot-is-authentic).
  So the server is faithful; the tooltip is just misleading.
- **DECISION IS OPEN (not a bug fix).** Two options, to be settled by player vote:
  - **(A) Keep as-is / retail** — KI benefits its own first shot (941 opener). This is
    what the original server did and what the working tree currently builds.
  - **(B) Match the tooltip** — defer KI's −10 to only the *next* 3s of shots (808
    opener, ramps to ~941). Fully implemented and **parked in `stash@{0}`**; can be
    restored if players prefer tooltip-accurate behavior.
  This is now a design/feel choice, not a correctness fix. See
  [§Proposed fix](#proposed-fix-defer-ki-to-after-the-triggering-shot) for the impl.
- Secondary "feels low" contributors (not bugs): KI's narrow >75%-health proc window;
  possibly base data values (base damage 450–540, attack rating 100) differing from
  the original patch.

## What was verified faithful (not the bug)

| Stage | Finding | How verified |
|---|---|---|
| Base weapon damage | From binary fire-mode setup, untouched | code read |
| `s_fDamageAdjustment` global scalar | **1.0 for players** (bot/difficulty-only, `combinedMultiplier != 1.0` gate) | `TgPawn__InitializeDefaultProps.cpp:213` |
| Buff stacking formula | `item × skill × self` layers; additive within a layer, multiplicative across | decompiled `GetBuffedProperty` `0x109d7ff0` + aggregator `0x109cd6f0` |
| Weapon mods (Targeting etc.) | Apply and stack correctly | player dummy test reconciles to <1 dmg across 8 combos |
| Target mitigation (`CalcCategoryProtection`/`DamageType`/`AttackType`) | **Intact native** — not reimplemented | grep: no server reimpl |
| Armor penetration (`GetAttackRating`) | **Intact native** + baked fire-mode data | grep: no server reimpl |

Because mitigation and penetration are intact native reading original inputs, any
"mitigation too high" is an **input** (data) question, not a formula bug.

## Recon ranged-damage kit (from `out/server.db`)

Property key: **214 = "Damage Modifier – Range"** (in the damage query expansion, so
buffs on 214 scale ranged damage). **155 = "Protection – Physical"**.

| Skill / source | egType | Effect | Class | Fires |
|---|---|---|---|---|
| Damage Increase: Ranged (528) | 261 Equip | prop214 **+6%** | 157 buff | passive ✓ |
| Sureshot (914) | 261 Equip | prop214 **+4%** (+proj speed) | 157 buff | passive ✓ |
| Recon Rifle Damage (598) | 261 Equip | prop214 **+10%** | 157 buff | passive ✓ (dummy-confirmed) |
| Super Sharpshooter (693) | 261 Equip **+10%** / **759** on-hit **+5%** | prop214 | 157 buff | passive ✓ / on-hit ✓ (log) |
| Killer Instinct (836) | **505** Hit-Situational | prop155 **−10 flat** on target, HP>75% gate | **80 base** | on-hit ✓ (log + A/B) |
| Targeting System (item, passive) | — | prop214 (+ power) | — | passive ✓ |
| Visual Scanner / Vulture Vision | — | (activated ability) | — | on-activate |

All type-261 passives install via `ReapplyCharacterSkillTree`; within the SKILL
layer they sum (RRD 10 + DmgInc 6 + Sureshot 4 + SS 10 = +30% on prop214), then
multiply against item/self layers. Verified faithful.

### On-hit dispatch works (log-proven)

`reconhit` channel, one PvP session (`out/logs/350/reconhit.txt`):

```
26 × nType=505 -> egId=16596   ← Killer Instinct fires every hit
14 × nType=759 -> egId=26675   ← Super Sharpshooter on-hit fires
```

UC's `TgDeviceFire.ApplyEffectType(T)` → reimplemented `GetSkillBasedEffectGroup`
(`0x10a6ef50`) pulls them from `s_SkillBasedEffectGroups` by type. Both on-hit recon
skills dispatch and apply.

### Killer Instinct debuffs the ENEMY (confirmed)

EG16596: type 505 Hit-Situational, prop155 −10 (Subtract), applied to the hit
target, gated on target HP > 75% (`situational 1271`, value 75) and recon-rifle
source (`required_skill 327`), 3s refresh, class 80 (base TgEffect → `ApplyToProperty`).

A/B on the target dummy setup:

| | shot 1 dmg | mitigated | shot 2 dmg | mitigated |
|---|---|---|---|---|
| KI ON | 994 | 777 | 1032 | 808 |
| KI OFF | 854 | 917 | 886 | 954 |

Pre-mitigation is identical KI on/off (994+777 = 854+917 = **1771**), only the
*target's* mitigation moved (−140). A shooter-side buff could not change the
target's mitigation → KI debuffs the enemy correctly. The ~140 swing is the intact
protection formula's response to −10 rating.

### The ramp is by design (not a bug)

Shot 2 > shot 1 on rapid follow-ups (994→1032, 854→886) because SS on-hit +5% and
KI −10 apply *after* the first hit lands, so only shots 2+ (within 3s) benefit.
First shot of an engagement is always base. Two shots >3s apart both compute at base
(observed 927 → 927). This means **in normal sniper pacing the ramp rarely triggers**
— a real contributor to "feels low," but original design.

## The 941→808 drop explained (NOT a bug)

Two shots reported as "same conditions" landed at 941 then 808, which looked like an
over-reversal. The `effects` capture (`out/logs/352/effects.txt`) disproves that.

**Shooter output is identical both shots:** `[CHECK-BUFF-MOD]` shows `585 → 1676.902`
pre-mitigation on both (lines 1018 & 1045). Nothing on the shooter side changed.

**The whole difference is target mitigation.** The `[SHIELD-MIT]` lines break it down:

| | pre-mit | prop-155 mitig. | prop-218 mitig. | dealt |
|---|---|---|---|---|
| Shot 1 | 1676.9 | **486** | 250 | 941 |
| Shot 2 | 1676.9 | **653** | 214 | 808 |

Physical mitigation jumped 486 → 653 (+167) because **Killer Instinct's −10 was
active on shot 1 and gone on shot 2** — the target's applied-effect list dropped
16 → 15 entries and KI (egId 16596) was removed between shots with
`m_fCurrent=10.00` (a **clean −10 reverse**, not an over-reverse).

So **808 is the true base; 941 was a legitimately KI-boosted shot.** Shot 1 had a KI
from a *prior* hit still inside its 3s window; shot 2 had none because KI expired and
the target was now <75% health (no re-apply — correct per the tooltip: "hit a target
with more than 75% health").

**Order matches retail — KI benefits its own triggering shot (by design).** Retail
`TgDeviceFire.uc` submits effects on impact in this order:

```
1410  SubmitHitEffects(..., 0, 505, 1271);   Killer Instinct (type 505, situational 1271)
1412  SubmitHitEffects(..., 1, 505, 1271);   KI (skill source)
1445  SubmitHitEffects(..., 0, 264);         weapon DAMAGE (type 264)
```

KI (505/1271) is submitted **before** the damage (264), so the −10 protection lands on
the target before the triggering shot's damage is mitigated → that shot benefits from
its own KI (provided the target is >75% at that instant). The server reproduces this
(log 352 shot 1 mitigated at the KI-reduced 486). So the *first* shot on a healthy
target is the strong one; once the target is <75%, KI's gate fails and shots drop to
base. Shot 1 > shot 2 is correct, not inverted.

(Note: SS on-hit type 759 *does* apply after damage — log 352 line 1027, `prop=214
slot 24→29`, after the shot-1 damage at line 1021. Different type, different phase;
that's the +5% ramp on rapid follow-ups.)

## Retail footage settles it: KI-on-first-shot is authentic

Original-server recon 1v1 footage (Scorpia, no active buffs, target = another recon),
decoded shot-by-shot:

| Fight | Shot | Observed dmg | Decode |
|---|---|---|---|
| 1 | opener | **941** | KI −10 applied to the triggering shot (base 808 − KI ≈ 941 after mitigation) |
| 1 | +3s later | **808** | KI + SS both expired → base |
| 1 | instant follow-up | **839** | SS on-hit +5% only (808 × 1.05 ≈ 839), KI gate now fails (<75%) |
| 2 | opener | **941** | KI on first shot again |
| 2 | within 3s | **977** | KI −10 **and** SS +5% both active (941 × 1.05 ≈ 977) |
| 2 | after 10s | **808** | everything expired → base |

All six data points reconcile only if **KI applies to its own triggering shot** — the
941 opener is impossible under tooltip-literal (defer) behavior, which would open at
808. This is the same order the server reproduces today. The earlier Ballista video
that looked like "KI ramps up after shot 1" was innate shred-stacking + SS, not KI
timing. **Conclusion: the pre-damage KI application is retail-faithful; do not treat
it as a bug.**

### Damage-model reconciliation (Scorpia example)

Ties the observed numbers to the pipeline end-to-end:

- **Offense**: base 585 → buffed **1676.9** pre-mitigation (includes weapon Output Mod
  585 × 1.75 ≈ 1023 displayed, plus the +30% SKILL-layer ranged stack). SS on-hit is
  +5% *additive within the SKILL layer* (→ ~839 dealt equivalent), KI is −10 physical
  *on the target* (→ 941 without SS, 977 with SS).
- **Target protection**: 39 physical (9 passives + **30 from HUMAN BASE ATTRIBUTES
  device 864 / eg3575 / eff13249, prop 155 +30, class 80** — retail-authentic, client
  tooltip confirms) + 21 ranged (7 × RRRRRR armor, 3 ranged each).
- **Mitigation** = 1 − (0.61)(0.79) ≈ **52%**, from the two sequential multiplicative
  layers (physical then ranged). Retail-authentic; **no systematic under-damage bug on
  the mitigation side.**

### Why recon damage still "feels low"

KI's proc window is narrow: it only helps a shot fired within 3s of a prior hit
*while the target is still >75% health*. A single sniper shot usually drops the target
below 75%, so the follow-up that would benefit finds the target already ineligible.
Net: most sniper shots land at base, and the −10 shred rarely applies to a meaningful
shot. Working exactly to spec — but the spec is why it feels weak. This is a
design/data question, not a code bug.

### Clean confirmation (optional)

Fire ONE shot at a full-HP target untouched for >3s, with `effects` + `reconhit` on.
`[SHIELD-MIT] prop=155` ≈ 653 (base, ~808 dmg) confirms KI applies after damage. A
value ≈ 486 (~941 dmg) on a verified-first shot would be the only thing that reopens
an ordering bug.

## Data values to diff against an original reference

Devices share the item ID space (device 2204 = item 2204 = "Ballista Mk I").

| Weapon | Base dmg (prop51, direct HP, calc Subtract) | Attack Rating | Innate on-hit debuff |
|---|---|---|---|
| Ballista Mk I / II / III | 450 / 495 / 540 | 100 | **−10 physical protection** (eg18972, class 80) |
| Scorpia Mk I / III | 450 / 540 | 100 | none |

- **Attack Rating 100** is the mitigation-critical value; if the original patch had
  higher AR, targets mitigated less and damage felt higher.
- **Ballista has an innate −10 protection shred; Scorpia does not.** A Ballista recon
  stacks −10 (weapon) + −10 (KI) = −20 on a healthy target. This innate shred is also
  a class-80 prop-155 debuff → **same over-reversal path as KI**, so on a Ballista the
  residual could be up to +20.

## Proposed fix: defer KI to after the triggering shot

**Status: OPTIONAL — PARKED IN `stash@{0}`, NOT in the working tree.** This is option
(B) above (tooltip-accurate). Retail footage proved the current pre-damage behavior is
authentic, so this is no longer a bug fix — restore it from the stash only if players
vote for tooltip behavior over retail. The working tree currently builds the retail
(KI-on-first-shot) behavior.

Note on the parked impl: the original design hooked `PostDamageHandler` @ `0x10a71cd0`
for the re-apply, but that native takes a **by-value 96-byte `FImpactInfo`** and
crashed through the Detours trampoline (logs 356/357 — crash in `CallOriginal` itself).
The working parked version instead re-applies from **`TrackDamagedPlayer` @ `0x109c0880`**
(clean int/ptr native, runs on the attacker after the hit). The `PostDamageHandler`
approach below is left for context but is the abandoned path.

Earlier status (superseded): IMPLEMENTED (1271-only) via a `PostDamageHandler` hook;
`PostDamageHandler` verified via vtable xref (`0x11706808`); intact native → hook
chains `CallOriginal`.

### Problem
KI = type 505 "Hit Situational." Client `ApplyHit` (`TgDeviceFire.uc`) submits 505
(lines 1410/1412) *before* the 264 damage (line 1445); `SubmitEffect` →
`ProcessEffect` applies it synchronously. The rebuilt `GetSkillBasedEffectGroup`
(`0x10a6ef50`) feeds KI into that pre-damage phase, so its −10 lands before the
bullet's mitigation is read. Log 354: first shot mitigates 486 (with KI) vs 653
(without) → the bullet eats its own debuff.

### Target behavior
Bullet's damage computed at un-debuffed protection; KI's −10 persists 3s for
subsequent bullets. Matches the two working references already in-game: the innate
Ballista shred (264, after its own damage) and SS on-hit (759, after damage).

### Scope
Only skill-sourced, on-hit, health-gated protection debuffs: **type 505 + situational
∈ {1270 "Health Below", 1271 "Health Above" = KI}**. Untouched: SS self-buff (759),
instant damage (264), reactive shields (1104), device-sourced innate shred (264,
already correct), backstab 505/509.

### Approach — suppress pre-damage, re-apply post-damage
1. **Suppress:** in reimplemented `GetSkillBasedEffectGroup`, skip returning groups
   whose `m_nSituationalType ∈ {1270,1271}`. This is the only path KI reaches
   pre-damage. (509/backstab and other 505s are unaffected — precise filter.)
2. **Re-apply:** hook **`PostDamageHandler` @ `0x10a71cd0`** (called at the tail of
   `TgEffectDamage.ApplyEffect`, args `pTarget, pInstigator(=shooter), Impact,
   fHealthChange, …`). **It is an INTACT native** (walks the target EffectManager's
   reactive-handler array `+0x468/+0x46C` for reflect/counter-attack), so the hook
   **must chain the original**. After chaining, pull the shooter's type-505 /
   situational-1270-1271 groups from `s_SkillBasedEffectGroups` and apply each to
   `pTarget` via `pTarget->ProcessEffect(group, false, shooter, Impact)` (mirrors UC
   `SubmitEffect`).

### Subtleties to get right
- **Chain the intact native** (reflect/counter-attack must still fire). Decompiled
  body: `decompiled/TgGame/UTgEffectDamage/UTgEffectDamage__PostDamageHandler/`.
- **Health gate on PRE-damage health.** KI's 1271 gate is "target >75% health *when
  hit*." At PostDamageHandler time health is post-damage → reconstruct
  `preHealth = pTarget.Health + fHealthChange` and gate on that, so a shot on a
  healthy target still arms KI even if that shot drops the target under 75%.
- **reqSkill match** (KI reqSkill 327 = recon rifles vs the firing weapon's skill) —
  replicate the `SubmitHitEffects` gate (lines 1562/1565). This also naturally scopes
  out DOT/AOE/reflect PostDamageHandler calls (their device isn't a recon rifle).
- **Skip category 963** (raw damage) — same gate the native itself uses.

### Faithfulness caveat
Deliberate deviation from the shipped-UC submit order; justified by the RE'd-server
premise + tooltip + the shred/SS timing references. Comment it as such in-code.

### Verification (after human build)
Repeat the Scorpia KI A/B: first shot should drop to **808** (base, == no-KI), shots
2–3 within 3s ramp to **~941**. Ballista+KI shot 1 → 808, shots 2–3 stack KI + shred.

### Files
- `TgEffectManager/GetSkillBasedEffectGroup/…cpp` — add situational {1270,1271} filter.
- New `TgEffectDamage/PostDamageHandler/…{hpp,cpp}` — chaining hook + re-apply (+ add
  the `.cpp` to `Makefile`).

## Other open items
- **Dweller exact values**: no *player* "Dweller" in the weapon table (only bot
  `YA_BOT_Desert_Dweller_Sniper_Rifle`). Need its in-game name / device id to diff
  base dmg / AR / innate shred.

## Current build state (2026-07-08)
- **Working tree = retail behavior** (option A). `GetSkillBasedEffectGroup.cpp` and
  `TrackDamagedPlayer.cpp` are both back to their clean/original forms; no KI deferral,
  no `[SKILL-ONHIT]` diagnostic logging.
- **Option B (tooltip) parked in `stash@{0}`** (`recon-sniper-dmg WIP moved off
  fix-decoy-visuals`): the suppression + `TrackDamagedPlayer` re-apply, plus the
  `reconhit` `[SKILL-ONHIT]` diagnostic. Restore from there if players vote for it.
- **Pending: player vote** on retail (A) vs tooltip (B) before any code change lands.
