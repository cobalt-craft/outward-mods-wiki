# Beastwhispering config reference

Every Beastwhispering setting lives in **`BepInEx/config/cobalt.beastwhispering.cfg`**, generated on
first launch. This page lists all of them, grouped by their `.cfg` section, with the default value and
what each one does. Values are read via `.Value`; edit the file with the game closed, or edit it live
and re-read it with the **`reloadcfg`** dev verb.

> **BepInEx has no config file-watcher.** A raw edit to the `.cfg` does nothing on its own — run
> `reloadcfg` (see [Dev verbs](./dev-verbs.md)) to re-read the whole file and re-apply, or relaunch.
> A handful of keys are baked into timers at pet-creation and only take effect on the next tame/reload
> even after `reloadcfg`; those are noted below. Some keys (Harmony patch toggles) are decided at
> startup and always need a relaunch.

Most systems ship a **kill-switch** (`Enable…`). Turning one off returns that feature to pre-feature
behaviour without disturbing the rest of the mod.

The command channel this mod polls is **`BepInEx/config/bw_cmd.txt`** (see [Dev verbs](./dev-verbs.md)),
and the per-creature data tables live as embedded defaults, each overridable by dropping a file of the
same name in `BepInEx/config/` (see [Data manifests](./data-manifests.md)).

### Example configuration

`BepInEx/config/cobalt.beastwhispering.cfg` — created on first launch. Excerpt below is a handful of
representative sections, not the whole file; the generated cfg is the only authority on its own size
(`cfgdump`, or `./scripts/forge cfg bw`, lists every section and key live). *A hardcoded section/key
count used to sit here and had drifted by roughly a third — don't reintroduce one.*

```ini
[Keys]
TameKey = F7
FeedKey = F8

[Systems]
InitialLoyalty = 55
LoyaltyGainPercent = 5
HungerSecondsPerDay = 7200
EnableItemHealing = true
UseSpeciesStats = true

[Temperature]
EnableTemperatureSystem = true

[Combat]
AttackDamage = 25
AttackInterval = 1.4

[Taming]
EnableTamingFoods = true
TameRadius = 15

[Bandage]
EnableBandageHealing = true
BandageItemIds = 4400010

[Cure]
EnableStatusCures = true

[DotAuras]
Enable = true
```

A full generated example lives at `tests/fixtures/config/cobalt.beastwhispering.cfg`, and the shared
cross-host overlay of the keys kept uniform is `config/shared/cobalt.beastwhispering.cfg.overlay`
(see `config/README.md`).

## [Keys] — keybinds

| Key | Default | Effect |
|---|---|---|
| `TameKey` | `F7` | Tame the nearest wild creature. |
| `FeedKey` | `F8` | Feed the pet the first inventory item its diet accepts. |
| `RecallKey` | `F9` | Recall the pet to your feet now (also re-forms a bodiless pet). The remedy for a stuck pet. |
| `SelfTestKey` | `F10` | Run the compute-layer self-test now. |
| `DiagKey` | `F12` | Dump a `[DIAG]` snapshot (location, vitals, combat, nearby AI, pet status) to the log. |

## [Pet] — the visible body

| Key | Default | Effect |
|---|---|---|
| `TameRange` | `15` | Search radius (m) to tame / re-form from. |
| `ModelYawOffset` | `180` | Degrees the creature model is rotated from the transform's forward. Tune if a species faces sideways. |
| `SpeciesYawOffsets` | *(empty)* | Per-species yaw overrides for rigs authored facing differently, as `Name=degrees` pairs. User-override layer over the shipped table. |
| `SpeciesPowerScales` | *(empty)* | Per-species loyalty **power scale** overrides, as `Name=scale` pairs (exact key beats the longest substring match). The user-override layer over the shipped `SpeciesPowerScale.txt` table (every shipped species is `1`); a species no layer names takes `1`. The scale multiplies the loyalty power **steps** — `0.5` means the top tier pays what a full species gets at 3 steps. Must be finite and in (0, 1] (refused with a `[POWER]` warning, falling to `1`). See `[Loyalty]`. |
| `FollowSpeed` | `4.5` | Pet base move speed (m/s) at full responsiveness. |
| `MinFollowSpeed` | `4.5` | Species-stats mode: minimum follow speed so a slow species can't lose a sprinting player. Combat chases run true species speed. |
| `LeashDistance` | `84` | Out-of-combat leash (m): further behind than this, the pet teleports to the nearest **off-screen** spot behind the camera (12.5–22.5 m back) and walks the rest in. Raised `30` → `42` → `84` on 2026-08-25 so it **matches `[Combat] CombatLeashDistance`** — one number in and out of combat, and the warp becomes a stuck-backstop rather than a leash you feel. This deliberately overrides the older “keep it under 50 (the zone re-place distance)” guidance: the owner-distance zone re-place rides this value up with it by design. The one backstop that does *not* follow it is the coverage-gap rescue for a pet whose goal is pinned away from you (a Stay spot, a scene spot, or a combat target left behind by an area change) — that keeps a flat 50 m floor so a stranded pet is still recovered. |
| `CatchUpSpeed` | `8` | Catch-up ceiling (m/s): more than ~5 m behind you, a following pet ramps up to this over the next ~4 m (like the vanilla cosmetic pets) so it closes the gap on foot instead of hitting the leash. Also the follow-speed **ceiling**: captured species speeds often exceed it (a Veaber records 18 m/s) and 8 is the run-animation ceiling — faster skates. Scales with the F2 `speedmult` dev slider. `0` = the old flat, unclamped follow speed. |
| `RestHealsPet` | `true` | Finishing a rest/sleep heals the pet in proportion to the sleep hours. |

## [Follow] — loafing near you when you stop

| Key | Default | Effect |
|---|---|---|
| `LoafDistanceMin` | `3` | Nearest distance (m) a stopped pet settles from you instead of standing on you (2 before 2026-08-20). A catching-up pet aims for the band's middle behind your direction of travel. |
| `LoafDistanceMax` | `5` | Farthest settle distance (4 before 2026-08-20). `0` = loafing off (settles on you). |
| `LoafDistanceRepick` | `3` | How far you must move before the pet picks a new loaf spot (stops it orbiting you). |

## [Systems] — simulation, loyalty, hunger, stats

| Key | Default | Effect |
|---|---|---|
| `InitialLoyalty` | `55` | Loyalty a freshly tamed pet starts with (0–250 since the 14-tier ladder; 55 = the Cautious floor). |
| `HungerSecondsPerDay` | `7200` | Game-seconds without feeding per "day" of loyalty decay — TWO real hours (1200 before the 2026-08-25 sustenance pass). Thirst is kept symmetric. `0` or negative disables hunger and decay entirely (a footgun). |
| `TempEscalateSeconds` | `300` | Sustained out-of-band seconds to worsen one comfort stage. Raised 30 → 300 on 2026-08-25: the SLOW RAMP is the weather lever, chosen over widening comfort bands (wide bands make pets weather-proof and strand the blanket / weather-food content). A short stop at a campfire never escalates; a long trek in a real extreme still does. Advances on real play seconds, so resting does not fast-forward it. |
| `TempRecoverSeconds` | `10` | Sustained in-band seconds to recover one comfort stage (recovery is deliberately faster than escalation). |
| `SpeciesDailyDecay` | `15` | Loyalty lost per day without feeding. |
| `LoyaltyGainPercent` | `5` | Percent of every **positive** loyalty gain (feeding, kill credit, first region crossing) that actually lands — the "slow, hard-won bond" lever. Nothing is lost to rounding: the leftover fraction is banked per pet and saved with it, so twenty preferred meals still deliver the ten loyalty one unscaled meal used to. Losses are **not** scaled by this. `100` = the original fast-bonding rates. |
| `SimTickSeconds` | `2` | How often (game-seconds) the pet systems advance. |
| `CastWatchdogWarnSeconds` | `5` | Log (diagnostic only) when the player's cast has been open this long. |
| `CastWatchdogClearSeconds` | `30` | Force-clear a wedged casting flag after this long. `0` = never. |
| `FeedHealAmount` | `10` | Pet HP healed by a feed when the food's diet entry has no `heal` of its own. |
| `EnableFoodHealthRecovery` | `true` | Feeding also starts a vanilla-style over-time Health Recovery regen (level 1–5 per food; taming chow always level 5). |
| `EnableItemHealing` | `true` | Feeding your pet a **healing item** heals it, by that item's **own** vanilla heal value — Life Potion and Great Life Potion, healing mixtures and any modded potion, with no data entry anywhere. Not food, so no diet lists it: any pet takes any potion. A pet already at full health refuses **without consuming**, and so does a pet whose wound is too small for the offer (a quarter of the item's value must be able to land — a scratch never burns a Great Life Potion). Damage effects are ignored, never applied. A diet food that happens to heal is still fed as a meal (authored heal only, never both); a potion that also cures — Great Life Potion cleanses Bleeding — heals **and** cures on one consume. Forensics: the `[ITEMHEAL]` log lines. |
| `SatiationFraction` | `0.5` | A pet fed within this fraction of its hunger day is fully satiated and refuses all food — with the 7200 s day, one hour of "not hungry" after an ordinary meal (two after a chow). `0` = never refuses. |
| `ChowSatietyMultiplier` | `2` | How much longer a SPECIES CHOW keeps the pet fed than an ordinary meal. `2` = satiated for two hours and hungry only after four, against one and two for a normal meal. Chow-specific (keyed off the taming registry), not a per-food setting; values below `1` are ignored. Stamped on the pet at the feed and saved with it, so a reload keeps it. |
| `EnablePassiveBuffs` | `true` | Grant the bond's passive stat buff to the player (per-species, scaled by loyalty tier). |
| `EnableBagPerk` | `true` | Grant the bond's flat backpack-capacity gift to the player's equipped bag. |
| `ShowPetStatusIcons` | `true` | Show the pet's hunger/bond/temperature state as indicator icons on the player HUD. |
| `HungryIconFraction` | `0.75` | Hunger fraction at which the Hungry icon appears (Starving appears at a full day). |
| `UseSpeciesStats` | `true` | The pet keeps the tamed creature's own combat stats (resistances, HP, attack, speed), captured at tame time and loyalty-tuned. Off = the flat config numbers apply. |
| `PersistPetHealth` | `true` | Persist the pet's current-HP fraction across loading screens and reloads. |
| `HealthLoyaltyFactor0` | `0.5` | Pet max-HP multiplier at loyalty 0 (lerps to the 100 value). |
| `HealthLoyaltyFactor100` | `1.5` | Pet max-HP multiplier at loyalty 100. |
| `DamageLoyaltyFactor0` | `1` | Pet damage/impact multiplier at loyalty 0. |
| `DamageLoyaltyFactor100` | `1` | Pet damage/impact multiplier at loyalty 100. |
| `DefenseLoyaltyFactor0` | `1` | Pet defense multiplier (resistances/protection/barrier) at loyalty 0. |
| `DefenseLoyaltyFactor100` | `1` | Pet defense multiplier at loyalty 100. |
| `SpeedLoyaltyFactor0` | `1` | Pet movement-speed multiplier at loyalty 0. |
| `SpeedLoyaltyFactor100` | `1` | Pet movement-speed multiplier at loyalty 100. |
| `ReleaseConfirmLoyaltyThreshold` | `0` | At or above this loyalty, releasing the pet must be **confirmed**: the first Release Pet cast (or `release` verb) only warns and arms a 30-second window; a second cast inside it — at least a second later, so a double-press or a batched command file cannot confirm itself — or an explicit `release confirm`, actually ends the bond. `0` = always confirm. A negative value disables the guard. |

`HungerSecondsPerDay`, `TempEscalateSeconds` and `TempRecoverSeconds` are baked into the sim at pet
creation — a live change applies only to the next tame/reload.

## [Temperature] — comfort, exposure, blankets

| Key | Default | Effect |
|---|---|---|
| `EnableTemperatureSystem` | `true` | Master switch for pet comfort (ambient sampling, discomfort stages, HP drain, death, blankets). |
| `EnableWeatherFoods` | `true` | Feeding a weather-resist consumable (potions, teas, water) grants the pet the player's hot/cold relief; all pets may drink water, and a **Weather Defense Potion** grants total weather immunity for its duration. |
| `PetDeathMode` | `Permanent` | What happens when exposure drains the pet to zero: `Permanent` (dies, bond deleted), `KnockedOut` (collapses/re-forms), `Disabled` (holds at 1 HP). |
| `SufferingDrainPerMinute` | `2` | Pet HP drained per minute (percent of max) while Suffering. |
| `CriticalDrainPerMinute` | `4` | Pet HP drained per minute (percent of max) while Critical (eased from 6 with the 2026-08-25 pass; `SufferingDrainPerMinute` deliberately stayed at 2). |
| `UneasyDecayMult` | `1.5` | Loyalty-decay multiplier while the pet is Uneasy. |
| `SufferingDecayMult` | `2` | Loyalty-decay multiplier while Suffering or Critical. |
| `BlanketRecipeDropChance` | `0.05` | Chance (0–1) that **each** blanket recipe scroll (Heating / Cooling) drops from a **world loot container** — chests (Chest, Ornate Chest, Trog Chest, Stash, Supply Cache) and the odd containers (Broken Tent, Hollowed Trunk, Junk Pile). Rolled independently per blanket, once per container when it fills. **Not** from creature kills, and not from corpse containers. `0` = craft-only. |

See [Temperature & blankets](./temperature-and-blankets.md).

## [Combat] — the pet fighting, damage scaling, vocals

| Key | Default | Effect |
|---|---|---|
| `EnableCombat` | `true` | Let the pet attack what you're fighting. |
| `AttackDamage` | `25` | Base damage per pet hit (used when not using species stats). |
| `AttackInterval` | `1.4` | Seconds between pet attacks. |
| `AggroRange` | `12` | How far the pet will engage an enemy you're fighting. |
| `AttackRange` | `2.6` | Melee reach for a pet attack. |
| `CombatLeashDistance` | `84` | Leash while actively fighting, so the pet can reach enemies engaged at range. Raised from `40` in 2026-08-24's owner-focus pass — it is what lets the pet keep a target `OwnerFocusRange` away instead of being warped home mid-charge — and again from `60` to `84` in the 2026-08-25 leash widening. It also raises the owner-distance zone re-place while the pet is fighting, so the two legs can't fight each other. |
| `AssistOnOwnerHit` | `true` | **The moment one of your attacks lands on an enemy, an engaged pet charges that enemy** rather than waiting for it to walk into `AggroRange`. This is what makes a pet useful to an archer: `AggroRange` is measured *pet*-to-enemy, and the game does not count something you shot at 30 m as an enemy you are "engaged" with until it aggros back and closes. The focus lasts 8 s and every further landed hit refreshes it, and it **outranks the pet's own self-defence** — the pet will leave whatever is biting it to go fight what you are shooting. Off = the older behaviour. |
| `OwnerFocusRange` | `60` | How far (m) the pet will charge to reach the enemy your attack just landed on. Keep it at or under `CombatLeashDistance` (which is `84`, so the shipped pair has room to spare). |
| `EnableSpecialAttack` | `true` | Allow firing the pet's Hunt as One signature attack. Requires `EnableCombat`. |
| `EngageRange` | `20` | Command Pet Engage: scan range for an enemy in front of you when you have no locked target. |
| `EngageConeDegrees` | `120` | Width of the "in front of you" cone the Engage scan uses. |
| `DisengageRunHomeSeconds` | `12` | After a fight ends far away, keep the relaxed leash this long so the pet runs home instead of teleporting. |
| `PetAttackVocals` | `true` | Play the species attack sound from the body when its bite animation fires. |
| `PetDamageScalePercent` | `100` | Global balance lever: scale every hit the pet deals, as a percent. `100` = unchanged. Covers all four pet damage paths. |
| `SpeciesDamageScales` | *(empty)* | Per-species overrides of the above, as `Name=percent` pairs (e.g. `Hyena=60`). Exact name wins, else longest match. |
| `UngovernedPetDamagePercent` | `50` | Pet damage while its owner has **not** learned Wild Unknown. `100` = no penalty. Damage only — travel is never gated. |

## [HuntAsOne] — the signature attack & your synced strike

| Key | Default | Effect |
|---|---|---|
| `EnableHuntAsOne` | `true` | Play the player's own synced strike alongside the pet's special. Requires `EnableSpecialAttack`. |
| `MeleeAnimation` | `Probe` | Cast-type name for the player's flourish with a melee weapon (or none). |
| `BowAnimation` | `PowerShot` | Cast-type name for the player's flourish with a bow. |
| `BonusDamage` | `20` | Flat bonus Physical damage the player's synced strike deals. `0` = animation only. |
| `BowShotDelay` | `0.5` | Seconds after the bow draw begins before the real arrow fires. |
| `EnableFoodHexes` | `true` | A species with a food-hex mapping inflicts hex build-up on its Hunt as One hit, chosen by its recent meals. |
| `HexMealWindow` | `3` | How many recent mapped meals drive the hex mix. |
| `HexBuildupPercent` | `20` | Hex build-up percent contributed per mapped meal. |
| `HonestHits` | `true` | Hunt as One strikes are failable — a block, dodge or whiff means that half deals nothing. |
| `ProcWeaponHitEffects` | `true` | The player's synced strike also fires the equipped weapon's own on-hit effects (enchantments, elemental procs), not just the flat bonus damage. |
| `EnableRangedSpecial` | `true` | A `Kind=Ranged` species (e.g. Mantis Shrimp) fires its own captured projectile on Hunt as One. |
| `BoltMaxFlightSeconds` | `6` | Backstop ceiling on waiting for a fired bolt's impact before ruling a miss. |
| `SyncSkillCooldownToPet` | `true` | Make the skill advertise the active species' real signature cooldown, so the hotbar countdown is truthful. |
| `BaseSkillCooldownSeconds` | `1` | The skill's own debounce cooldown when there's no species cooldown to sync to. |

## [Loyalty] — the 14-tier ladder, locks, and power from the bond

Loyalty runs 0–250 over fourteen tiers (see [Loyalty & bonds](./loyalty-and-bonds.md)). The whole
ladder — each tier's floor, payout level, lock flag and power step — is one table you can retune
live with `LadderOverride` + `reloadcfg`. The upper tiers (above Devoted) each pay one **power
step** to the pet's effective stats, applied after the 0–100 loyalty stat lerp. The per-step amounts
are the six knobs below; a species' `powerScale` manifest axis (or `[Pet] SpeciesPowerScales`)
scales the steps. This is where the old **species relic** bonuses moved: relic scaling ships off and
the relics are dormant.

```ini
[Loyalty]
LadderOverride =
EnableLocks = true
EnableRelicScaling = false
HpMultPerStep = 1.05
DmgMultPerStep = 1.054
ResistPerStep = 1
ProtPerStep = 1
BarrierPerStep = 1
StatusResPerStep = 1
```

| Key | Default | Effect |
|---|---|---|
| `LadderOverride` | *(empty)* | Retune the whole 14-tier loyalty ladder without a rebuild: exactly 14 comma-separated rows of `floor[:payout[:lock[:power]]]` in tier order (Gone, Broken, Fraying, Guarded, Cautious, Steady, Trusting, Devoted, Unshaken, Boundless, Sworn, Fierce, Mythic, Eternal); an omitted field keeps the shipped row's value. Row 0's floor must be 0, floors strictly ascend (top ≤ 250), payouts 0–8 never fall, lock is 0/1, power ≥ 0. The shipped table is `0:0:0:0,1:1:0:0,15:2:0:0,40:3:0:0,55:3:1:0,75:4:0:0,90:4:1:0,100:4:0:0,125:5:1:1,150:5:0:2,175:6:1:3,200:7:0:4,225:7:1:5,250:8:1:6`. A defective string is refused loudly (`[LOYALTY]` warning) and the shipped table stays. Live via `reloadcfg`; the `[LOYALTY] ladder` boot/reload line prints the live table. |
| `EnableLocks` | `true` | Loyalty **locks**: once a pet reaches a lock tier (Cautious, Trusting, Unshaken, Sworn, Mythic, Eternal on the shipped ladder) its loyalty can decay down *to* that tier's floor but never below it — a Trusting pet can never fray back to Broken, and a locked pet can never be abandoned. The floor is saved with the pet. `false` = loyalty decays freely; an already-earned floor stays in the save, inert, for a later re-enable. Dev: `setloyalty N` respects the floor (`setloyalty N force` lowers it); `setfloor N` sets it. Live via `reloadcfg`. |
| `EnableRelicScaling` | `false` | Do species-relic stacks (Pearlbird's Courage, Leyline Figment, Metalized Bones) still buff the pet? Ships **off**: their per-stack bonuses moved onto the upper loyalty tiers. While off, relics are **dormant** — the pet refuses its relic **without consuming it**, already-eaten stacks stay on the save but pay nothing, and the Companion tab's *Relics* row reads "dormant". Turn on to restore the old relic feed **on top of** the loyalty power. |
| `HpMultPerStep` | `1.05` | Max-HP multiplier per loyalty power step, compounding (`1.05` = ×1.34 at the 6-step top tier). Must be finite and > 0. |
| `DmgMultPerStep` | `1.054` | Damage multiplier per step, compounding, on every damage **type** — Impact is deliberately untouched. `1.054` = ×1.37 at the top tier. |
| `ResistPerStep` | `1` | Flat all-resistances points per step (clamped to −100..100; a species weakness shrinks toward 0). `1` = +6 at the top tier. |
| `ProtPerStep` | `1` | Flat ProtectionAll per step. |
| `BarrierPerStep` | `1` | Flat Barrier per step. |
| `StatusResPerStep` | `1` | Flat all-status build-up resistance per step, under the same sub-100 ceiling every other contributor respects. |

All six apply on the next sim tick — no relaunch. Forensics: the `[POWER]` boot lines report the
power-scale table and whether relic scaling is on.

## [Synergy] — the Hunt as One stacking buff

| Key | Default | Effect |
|---|---|---|
| `EnableSynergy` | `true` | Landing both halves of Hunt as One grants a stack of Synergy (+all-damage for player and pet). |
| `PercentPerStack` | `0.5` | All-damage bonus percent per stack. |
| `MaxStacks` | `8` | Stack cap. |
| `DurationSeconds` | `300` | Lifespan of the stacks (a new grant refreshes all). |
| `WindowSeconds` | `6` | How long after a cast the two halves may land and still count. |
| `RequireSameTarget` | `false` | Require the player's hit to be on the pet's target to count. |
| `AdoptPlayerArrows` | `true` | Bow parity: while a Hunt as One window is open, **any** arrow you fire can be the player half — not just the one Hunt as One force-shoots for you. Judged at impact by the same honest rules, and it carries no bonus-damage rider. |
| `BowStacksPerGrant` | `2` | Stacks granted when the player half was an **arrow**, vs the 1 a melee half grants. Bows are the harder half to land (flight time, terrain, no second swing inside the window), so they pay double. Still clamped to `MaxStacks`. Set `1` for no bow bonus. |
| `PlayerMeleeRangeMeters` | `3.5` | How close you must be to the pet's target for the synced melee strike to connect. |
| `PlayerMeleeConeDegrees` | `100` | Front-cone width the target must be inside for the synced strike to connect. |
| `PetMeleeRangeMeters` | `4.5` | How close the pet must be to its target for its melee special to connect. |
| `ResistCap` | `50` | The ceiling (raw all-resistance points) the pet's Synergy resistance bonus approaches at full loyalty: `resist = (loyaltyTier/4) x ResistCap x (1 - e^-(stacks x speciesMultiplier)/12.5)`. Loyalty scales the ceiling linearly; stacks approach it with diminishing returns, so the bonus can never reach this number. Raising it scales the whole axis without changing how fast any species climbs. Applies on the next sim tick; forensics: `synergydump`. |
| `ResistMultiplierDefault` | `0.15625` | The species multiplier used for any pet with no `synergyResist` entry of its own — i.e. almost every pet; this **is** the global backfill. The shipped value is 5/(4x8): exactly +5 at the top loyalty tier and the 8-stack cap. Must be finite and > 0 (refused with a `[SYNRESIST]` warning, falling back to the default). Per-species overrides: the embedded `SpeciesSynergyResist.txt` table, then `[Pet] SpeciesSynergyResistMultipliers`, which wins per key — see [data manifests](./data-manifests.md). |
| `FlashOnGrant` | `true` | Play the Pommel Counter success flash on **both** you and your pet on every Synergy grant or refresh — the orbit on your main-hand weapon (a default shape if you are unarmed) and a one-shot on the pet's body. Replicated in multiplayer. Purely cosmetic; the stack and bonus are unaffected. Also silent when CompanionKit `[PetFx] EnablePetFx=false`. Live via `reloadcfg`. |
| `ObserverPatch` | `false` | Diagnostic only (not a gate): log the player's real weapon connects while a Synergy attempt is open, to compare against the judged verdict. Decided at startup — relaunch to change. |

`BepInEx/config/cobalt.beastwhispering.cfg` excerpt — turning the counter flash off while keeping the buff:

```ini
[Synergy]
EnableSynergy = true
FlashOnGrant = false
```

## [ForTheKill] — the Synergy spender

| Key | Default | Effect |
|---|---|---|
| `EnableForTheKill` | `true` | Consume all Synergy stacks for one joint execute strike plus the species' debuff. |
| `CooldownSeconds` | `35` | For the Kill cooldown. Every cast burns it. |
| `BaseDamageMult` | `1.5` | Base multiplier on the joint strike before per-stack scaling. |
| `DamagePerStackPercent` | `35` | Extra damage percent per stack consumed. |
| `EnableKillFavor` | `true` | A kill by the execute grants the player a loyalty-tiered stat buff (the species' `killBuff`). |
| `KillFavorDurationSeconds` | `300` | How long that buff lasts. |
| `EnableLeap` | `true` | The Predator's-Leap flourish. Kill-switch for **all three** of its faces: the player's arc, the pet's arc, and the bow-mode ground stomp (below). **SENDER-SIDE in co-op:** it gates whether *your* cast produces a leap or a stomp at all, so turning it off silences everything you cast. It does **not** gate what you are shown — another player whose own `EnableLeap` is on still sends `bw.ftk.leap` / `bw.ftk.stomp`, and your machine still replays their arc and crater. A receiver-side mute is deliberately not wired (static analysis 2026-08-23 AF4-8); if you want a quiet screen in co-op, everyone in the room needs the setting off. |
| `LeapRangeMeters` | `5.5` | Cast at a target within this range and the leaper arc-jumps to it. Beyond it (or closer than 1m) nobody hops. |
| `LeapDurationSeconds` | `0.45` | Airtime, clamped down to the pet's strike windup so both leaps land before the hit is judged. |
| `LeapApexMeters` | `1.5` | Height of the arc at its midpoint. |

**Melee vs bow.** With a melee weapon, casting For the Kill arcs *both* the player and the pet onto
the target, each landing with a crunch and a stone-debris crater. **With a bow the player does not
leap at all** — the cast already draws a power shot and looses a real, damage-scaled arrow, so the
archer holds their ground and that crater + crunch plays under *their own feet* instead, and only if
the arrow lands an unblocked hit on an enemy. The pet still arcs either way. (With
`[HuntAsOne] HonestHits = false` — the legacy guaranteed-hit kill-switch — no arrow is tracked, so
bow casts keep the old leap.)

`BepInEx/config/cobalt.beastwhispering.cfg`:

```ini
[ForTheKill]
EnableLeap = true
LeapRangeMeters = 5.5
LeapDurationSeconds = 0.45
LeapApexMeters = 1.5
```

## [Brace] — the tank counterattack stance (Armored Hyena)

| Key | Default | Effect |
|---|---|---|
| `EnableBrace` | `true` | A `Kind=Brace` species's Hunt as One opens a counterattack window instead of a strike. |
| `WindowSeconds` | `5` | How long the braced window stays open. |
| `PerAttackerOnce` | `true` | Counter each attacker only once per window. `false` = counter every eligible hit. |
| `NegateCounteredHit` | `true` | A countered hit deals zero damage to the pet. |
| `RiposteImpact` | `60` | Impact (knockback) the riposte carries. |
| `CueVolume` | `0.5` | Volume (0–1) of the parry cue at brace-enter. `0` = none. |
| `EnableTaunt` | `true` | Entering the stance pins the pet's current target's aggro onto the pet. The pin lasts **1 second at minimum loyalty rising to 5 seconds at maximum**, and fires once per Hunt as One cast, with an on-screen line (*"… draws the enemy's fury with an infernal growl!"*). Species without a taunt entry in their data are unaffected. |

## [Evade] — the Hyena's dodge

Ships **off**. A species with an evade entry (`SpeciesEvade.txt` — the Hyena: 10% at the bottom of the
loyalty ladder, 18% at Eternal) can dodge an incoming melee, arrow or bolt: the hit does no damage and
the pet plays its dodge animation, sound and a short hop away from the attacker. Area blasts and
damage-over-time ticks are never dodged. In co-op the host's settings decide for every pet in the room.

| Key | Default | Effect |
|---|---|---|
| `EnableEvade` | `false` | Turn the dodge on. `false` = no roll, no animation — the feature is dark. |
| `HopMetres` | `3.4` | How far the pet hops away from the attacker. The path stops at the first wall or navmesh edge (a hop shorter than 0.5 m is skipped); no walkable ground there = no hop (the dodge still counts). |
| `HopSeconds` | `0.3` | How long the hop takes. |
| `CooldownSeconds` | `3` | After a dodge, the pet cannot dodge again for this long. |
| `ReengageHoldSeconds` | `0.6` | After the hop the pet holds its ground for this long before closing on the enemy again (0–3; `0` = re-close immediately and skip the face-the-attacker turn). |

Per-species override without a rebuild: `BepInEx/config/SpeciesEvade.txt`, one `Species=chance,chanceAtMax,Trigger`
line each (e.g. `Hyena=100,100,Dodge` dodges everything), then `reloadevade`. Forensics: `evadedump`; `forceevade [n]` makes the next n eligible hits dodge for certain.

## [IdleBreak] — the Pearlbird's peck and preen

Ships **on**. A species with an idle-break entry (`SpeciesIdleBreaks.txt` — the Pearlbird: three peck/preen
clips) plays one of them when you stand still: the first a few seconds after the bird settles, then every
so often while you keep idling. Moving, an attack or a knock cancels it. Purely cosmetic — nothing to do
with combat, and in co-op only you see your own bird's pecks (v1).

| Key | Default | Effect |
|---|---|---|
| `Enable` | `true` | Turn idle breaks on. `false` = the bird just stands. |
| `StartSecondsMin` / `StartSecondsMax` | `2` / `8` | How long after the bird settles (and you are idle) the first break fires — a random point in this window. |
| `RepeatSecondsMin` / `RepeatSecondsMax` | `5` / `15` | Gap between breaks while you keep standing still. |

```ini
[IdleBreak]
Enable = true
StartSecondsMin = 2
StartSecondsMax = 8
RepeatSecondsMin = 5
RepeatSecondsMax = 15
```

Per-species override without a rebuild: `BepInEx/config/SpeciesIdleBreaks.txt`, one
`Species=Trigger,CountParam,Count` line each, then `reloadidlebreaks`. Forensics: `idlebreakdump`;
`idlebreak [n]` plays variant `n` right now. Idle breaks while following ride the loaf feature (`[Follow] LoafDistanceMax` must be above 0); on a Stay spot they fire regardless.

## [SkillEcho] — the pet's bonus strike on your weapon skills

| Key | Default | Effect |
|---|---|---|
| `EnableSkillEcho` | `true` | Casting a classic weapon skill (Puncture, Moon Swipe, etc.) makes the pet strike your target. |
| `EchoDamageMult` | `1.25` | Default echo damage as a multiplier of the pet's base damage. |
| `EchoImpactMult` | `1.25` | Default echo impact as a multiplier of the pet's attack impact. |
| `EchoWindupSeconds` | `0.4` | Delay between the echo's animation and the hit resolving. |
| `EchoCooldownSeconds` | `1` | Minimum seconds between echoes. |
| `EchoRangeMeters` | `10` | The pet must be within this range of the target or the echo skips. |
| `EchoWhilePassive` | `false` | Fire even while the pet is in Disengage stance. |
| `CueOnPet` | `true` | Play the cue VFX on the pet (`true`) or the player (`false`). |
| `CueStatusName` | `Rage` | Status whose FX prefab is borrowed for the cue (never applied). |
| `CueSeconds` | `2` | How long the cue VFX lives. |

## [Gifts] — the Gift skill & feather fletching

| Key | Default | Effect |
|---|---|---|
| `EnableGiftSkill` | `true` | The Gift skill: the pet gives the player a species-defined gift. |
| `GiftCooldownSeconds` | `600` | Gift skill cooldown (10 min). A no-pet cast refunds it; a "nothing" roll burns it. |
| `EnableFeatherFletching` | `true` | Pearlbird feathers enchant arrows in place with quality-tiered "Pearl Fletching". Freshly fletched arrows re-stack with any arrows of the same type and the same fletching, including the stack in your quiver; differently-fletched stacks never combine. |
| `FletchArrowsPerFeather` | `15` | Arrows one feather fletches, gathered across every stack of that arrow type in your pouch and equipped bag (so arrows that will not stack in the crafting menu still cost one feather). A bigger stack is split; `0` = uncapped. |
| `FletchNameSuffix` | `true` | Fletched arrow stacks show their bonus in the display name (e.g. "Iron Arrow (+20%)"). |

## [Scent] — the pet's environment sense

| Key | Default | Effect |
|---|---|---|
| `EnableScentSense` | `true` | A species with a senses entry periodically noses out nearby interesting item spawns and indicates them. |
| `SniffIntervalSeconds` | `5` | Seconds between scent scans. |
| `SkipEmptyGatherables` | `true` | Ignore already-harvested gathering spots. |
| `NorthYawOffsetDegrees` | `0` | Compass correction for the direction bucketing. |
| `IndicatePoint` | `true` | On a fresh scent the pet stops, faces the source (the hunting "point"), and barks. |
| `PointSeconds` | `3` | How long the pet holds the point. |
| `IndicateStatusIcon` | `true` | Show a "Scent Trail" buff icon while the pet holds a scent (also needs `ShowPetStatusIcons`). |
| `IndicateToast` | `true` | On a fresh character-kind alert, an on-screen toast names the scent and its compass direction. |

## [Sigils] — mage sigils and the pet

| Key | Default | Effect |
|---|---|---|
| `EnableSigilSynergies` | `true` | A species with a sigils entry changes its Hunt as One hit while the pet stands in a mage sigil. |
| `EnablePetSigils` | `true` | A species with a sigil of its own may lay that circle at its own feet. |
| `PetSigilScale` | `0.75` | Size of the pet-laid circle relative to a player-cast sigil. |
| `PetSigilCooldownSeconds` | `60` | Minimum seconds between pet-laid sigils. |

## Single-switch systems

| Section · Key | Default | Effect |
|---|---|---|
| `[Gear] EnableGearEffects` | `true` | A worn item with a gear entry alters the active pet (e.g. a Pearlbird Mask boosts loyalty gain). |
| `[BuffFoods] EnableBuffFoods` | `true` | Feeding a buff food grants a temporary damage buff. It fills the belly too only if the species' diet also accepts that item (then one item does both); otherwise it is buff-only. |
| `[Scavenge] EnableScavengeBonus` | `true` | A species's listed loot containers roll extra times on first open, by loyalty tier. |
| `[HUD] EnableHealthHud` | `true` | Show a simple pet-health bar (top-left) while a live pet exists. |
| `[DotAuras] Enable` | `true` | While your pet is burning, poisoned, bleeding or plagued it visibly wears that status's own vanilla effect, replicated to every machine. Without it the effect is invisible — the pet takes the damage on its hidden combat body, whose renderers are switched off, so it just loses health for no apparent reason. Several can show at once (unlike weapon infusions, damage-over-time effects stack). Off = every worn aura clears within one tick, locally and remotely. Forensics: `dotauradump`. |

## [Skills] — the skill-tree gates

| Key | Default | Effect |
|---|---|---|
| `ScatologyGatesScrolls` | `true` | Recipe scrolls only drop once someone in the session has learned Scatology. |
| `BeastOfBurdenCapacity` | `10` | Total flat carrying capacity Beast of Burden grants while you have a pet, **split 70/30: 70% onto the pet's own inventory, 30% onto your pouch** (at the default: **+7 pet / +3 pouch**). It adds nothing to your equipped backpack. `0` = the skill grants nothing anywhere. |
| `EnableCommunionGate` | `true` | The bond's passive player buffs require the Communion skill. (This default withholds buffs on existing saves until Communion is bought.) |

Beast of Burden is **one knob with a fixed 70/30 split** — there is deliberately no separate pet key
and pouch key. In `BepInEx/config/cobalt.beastwhispering.cfg`:

```ini
[Skills]

## Total flat carrying capacity the Beast of Burden passive grants while the player has a pet ...
# Setting type: Single
# Default value: 10
BeastOfBurdenCapacity = 10
```

That grants **+7** on the pet's own inventory (added on top of its species carry weight — see
[Data manifests](./data-manifests.md), `SpeciesCarry.txt`) and **+3** on your pouch. Set it to `12`
for +8.4/+3.6, or to `0` to switch the skill's capacity off entirely. Your equipped backpack is not
affected by this skill at all; the backpack bonus some pets grant is the separate per-species
`BagCapacity` gift in `SpeciesBuffs.txt`, gated by `[Systems] EnableBagPerk`.

> A changed default never migrates into a `.cfg` that already exists. An install generated before
> this change still holds `BeastOfBurdenCapacity = 5` (which would read as +3.5 pet / +1.5 pouch) —
> set it to `10` by hand, or let `scripts/sync-config.py` patch it from
> `config/shared/cobalt.beastwhispering.cfg.overlay`, where it is pinned.

See [Skills](./skills.md).

## [Wards] & [Lantern] — sharing player spells with the pet

| Key | Default | Effect |
|---|---|---|
| `[Wards] EnableWardShare` | `true` | The pet's anchor mirrors your Mana Ward (Force Bubble) and receives Gift of Blood's ally regen. |
| `[Wards] WardPollSeconds` | `0.25` | How often the player's ward statuses are polled. |
| `[Wards] ManaWardStatusNames` | `Force Bubble,ForceBubble` | Status-identifier candidates for Mana Ward's status. |
| `[Wards] GiftOfBloodStatusNames` | `Gift of Blood,GiftOfBlood` | Candidates for the caster-side Gift of Blood status. |
| `[Wards] GiftOfBloodAllyStatusNames` | `Gift of Blood Ally,GiftOfBloodAlly` | Candidates for the ally-side regen status. |
| `[Lantern] EnableLanternShare` | `true` | A copy of your summoned Runic Lantern floats above the pet while yours is alive. |
| `[Lantern] LanternPollSeconds` | `0.25` | How often the player's statuses are polled for the lantern. |
| `[Lantern] LanternStatusNames` | `Runic Lantern Amplified,Runic Lantern,RunicLantern` | Status-identifier candidates for the lantern (base and amplified are separate statuses). |
| `[Lantern] LanternUpOffset` | `0.5` | How far above the pet the mirrored lantern floats (m). |
| `[Lantern] LanternForwardOffset` | `0.3` | How far ahead of the pet it floats (m). |

## [Bandage] — applying a bandage to the pet

| Key | Default | Effect |
|---|---|---|
| `EnableBandageHealing` | `true` | On a bandage item the right-click "Feed" inventory action becomes "Bandage <pet>": pressing it puts the vanilla heal-over-time status on the pet's anchor (the same one a bandaged player gets), consumes the bandage, and grants the player no buff. Off = a bandage is an ordinary item again (the action falls back to "Feed", which a bandage refuses). |
| `BandageItemIds` | `4400010` | ItemIDs treated as a "bandage" for the apply-to-pet action, comma-separated (vanilla Bandages = `4400010`). Add a modded bandage's id to make it applicable too. |
| `BandageStatusNames` | `Bandage` | Status-identifier candidates for the heal status a bandage grants, comma-separated, first that resolves wins (identifiers are asset data; `statusdump`/`bandagedump` list what resolved). |

## [Cure] — curing the pet's status effects

| Key | Default | Effect |
|---|---|---|
| `EnableStatusCures` | `true` | Feeding your pet a remedy cures it, exactly as using that remedy cures you: the item's *own* vanilla cleanse effect is applied to the pet's body. Antidote clears every tier of poison at once (it cleanses the Poison type, not one status), Bandages clear bleeding as well as healing, and Panacea / Hexes Cleaner / Great Life Potion / Myconic Cleanser / Grilled Marshmelon and friends all work for the same reason — nothing is special-cased. A remedy is not food, so no species diet has to list it: any pet takes any cure. An item that would cure nothing is never consumed. A cure that rides a real meal is applied on the same single consume. Off = remedies are ordinary items again. Forensics: `curedump`. |
| `WaterCuresStatuses` | `Burning,Burn` | **Fallback only.** A drink of plain water douses a burning pet exactly as it douses a burning player, and that is vanilla's own doing: every water type's drink effects live on the game's shared water dispenser (which is why no water *item* appears to carry a cleanse) and include a cleanse of the Burning status tag — so Immolate rides along, and the hot-weather temperature states, which are not on that tag, do not. We run those very components, so nothing here is modelled. This key is consulted only if that dispenser cannot be read yet, and then these comma-separated status identifiers are matched by name instead — all of them, not first-wins, since a fire source may apply either spelling. Discovery: `statusdump`; `curedump` reports which of the two paths is live. |

## [Anchor] — the invisible combat body

The pet's real combat body is an invisible ally Character ("the anchor") that absorbs enemy aggro and
damage while the visible body puppets it. Most of these are for debugging.

| Key | Default | Effect |
|---|---|---|
| `EnableAnchor` | `true` | Spawn the invisible ally so enemies can target and damage the pet. |
| `AnchorInvisible` | `true` | Hide the anchor's model. Off = watch the ally directly while debugging. |
| `AnchorShowHealthBar` | `false` | Show the anchor's floating health bar. |
| `AnchorLinkSummonSlot` | `true` | Register the anchor as the player's summon (rides owner teleports; conflicts with a real Conjure ghost). |
| `HideSummonIcon` | `true` | Hide the synthesized "Summoned Ghost" HUD icon for the anchor. |
| `AnchorLeashDistance` | `28` | **Fallback only — it does not govern an ordinary pet.** Teleport the invisible anchor to the player when it falls this far behind (out of combat; `CombatLeashDistance` applies while fighting). At the shipped `GlueMode = Always` the anchor is welded to the visible body every frame, so its position is not its own and this leash is skipped — the body's `[Pet] LeashDistance` (84) owns recovery and drags the anchor with it. The skip is deliberate: 28 sits far *below* 84, so applying it literally under the weld would pull the anchor home while the body is still legitimately out, splitting the pet in two. It bites only under `GlueMode = Off` / `Combat` (out of combat) and for a bodiless remote proxy anchor. Raised from `20` in the 2026-08-25 leash widening (+40%). |
| `AnchorRespawnSeconds` | `60` | How long a downed pet stays gone before re-forming. |
| `AnchorBaseHealth` | `110` | Base for the pet's combat HP (×1.5 at loyalty 100, ×0.5 at loyalty 0). |
| `AnchorDealsDamage` | `false` | Let the anchor's invisible weapon deal damage. Off = the visible-combat system owns damage. |
| `AnchorSpeciesVoice` | `true` | Species-correct hurt/death cries + silencing of the ghost's stock audio. |
| `GlueMode` | `Always` | How the anchor is position-coupled to the visible body. `Always` = welded every frame. |
| `GlueOffsetBehind` | `0` | Weld offset (m) of the invisible anchor along the body's facing: positive = behind, negative = ahead, 0 = dead-centre. Was `0.3`; enemies measure their stop distance to the anchor, so an offset onto the hindquarters had them stepping into the model to reach it. Range −0.5…1. |
| `UnifyTargets` | `true` | Keep the anchor and pet locked on the same enemy. |
| `AnchorPlayerCollision` | `PassPlayer` | How the anchor collides with players: `PassPlayer` (can't shove you), `Block` (pre-fix), `Phantom` (blocks nothing). |

## [PetPanel] — Companion-tab controller nav and naming

| Key | Default | Effect |
|---|---|---|
| `EnableControllerNav` | `true` | Right stick scrolls the active half of the Companion tab; triggers switch halves. |
| `StickScrollSpeed` | `1.2` | Scroll speed at full stick deflection. |
| `StickDeadzone` | `0.2` | Right-stick deadzone before scrolling. |
| `EnableRename` | `true` | Let the Companion tab's header be clicked (or opened with the pad's top-right face button) to name your pet. Off = the header is a plain label and your pet always shows its species. |

```ini
[PetPanel]
EnableControllerNav = true
StickScrollSpeed = 1.2
StickDeadzone = 0.2
EnableRename = true
```

## [Taming] — the player taming loop

| Key | Default | Effect |
|---|---|---|
| `EnableTamingFoods` | `true` | The taming loop: recipe-scroll drops, cooking chow, and taming a wild one by using it nearby. |
| `TameRadius` | `15` | How close a wild creature must be for using its taming food to tame it. |
| `TameRecheckGraceMult` | `1.5` | Slack on `TameRadius` for the re-check after the eat animation, so a creature that drifted a step away during the animation still counts. |
| `RecipeDropChance` | `0.33` | Global chance (0–1) a tameable creature drops its recipe scroll on death, ×its per-species chance. Roughly one scroll per three tameable kills. |

> **A changed default does not reach an existing install.** BepInEx writes a `.cfg` once and never
> migrates a new default into it, so a game that has already generated
> `BepInEx/config/cobalt.beastwhispering.cfg` keeps whatever number that file holds. Edit the key by
> hand, or delete the file and let it regenerate.

## [Maren] — the trainer NPC

| Key | Default | Effect |
|---|---|---|
| `EnableMaren` | `true` | Register Maren the trainer with StoryKit. Off = she never appears (relaunch to change). |
| `MarenScene` | `CierzoNewTerrain` | Scene build name Maren stands in. |
| `MarenPosX` / `MarenPosY` / `MarenPosZ` | *(Cierzo spot)* | Maren's spawn position. Re-stamp with the `marenhere` verb. |
| `MarenRotY` | *(Cierzo facing)* | Maren's facing (yaw degrees). |

## Diagnostics & maintenance

| Section · Key | Default | Effect |
|---|---|---|
| `[SelfTest] RunSelfTestOnLoad` | `false` | Run the compute-layer self-test on load. |
| `[Diag] DiagIntervalSeconds` | `60` | Auto-dump a `[DIAG]` snapshot on this cadence. `0` = off. |
| `[Diag] DiagRadius` | `30` | Radius the `diag` snapshot scans for nearby AI. |
| `[Diag] CastDiagPatches` | `false` | Re-arm the cast-pipeline tracer (relaunch to change). |
| `[Diag] MusicReconPatches` | `false` | Re-arm the music-recon taps (relaunch to change). |
| `[Harvest] UnloadUnusedAssetsMode` | `EveryN` | When to run the post-harvest asset purge (`Always` / `EveryN` / `Off`). |
| `[Harvest] UnloadEveryNHarvests` | `5` | With `EveryN`: purge only every Nth harvest. |
| `[Harvest] FlushTerrainAfterPurge` | `true` | Re-arm terrain render data after a purge. |

> **`[Expedition]` is a legacy section.** Its keys (`CaptureOnSceneEntry`, `AutoWarmAtBoot`,
> `AlwaysWarmSpecies`) live in **[DonorKit](../../kits/donorkit.md)**'s own config,
> `BepInEx/config/cobalt.donorkit.cfg`; the copies here are read once to migrate a customised value
> and then ignored. Edit DonorKit's config instead.

## See also

- [Beastwhispering overview](./README.md)
- [Dev verbs](./dev-verbs.md) — the `reloadcfg` verb and the command channel
- [Skills](./skills.md) · [Combat & companion](./combat-and-companion.md) · [Temperature & blankets](./temperature-and-blankets.md)
- [Data manifests](./data-manifests.md) — the per-creature data behind these systems
- [Mods index](../README.md)
