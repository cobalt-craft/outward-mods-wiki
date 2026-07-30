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

`BepInEx/config/cobalt.beastwhispering.cfg` — created on first launch. Excerpt (representative
sections; the file has 31 in all):

```ini
[Keys]
TameKey = F7
FeedKey = F8

[Systems]
InitialLoyalty = 50
LoyaltyGainPercent = 5
HungerSecondsPerDay = 1200
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
| `FollowSpeed` | `4.5` | Pet base move speed (m/s) at full responsiveness. |
| `MinFollowSpeed` | `4.5` | Species-stats mode: minimum follow speed so a slow species can't lose a sprinting player. Combat chases run true species speed. |
| `RestHealsPet` | `true` | Finishing a rest/sleep heals the pet in proportion to the sleep hours. |

## [Follow] — loafing near you when you stop

| Key | Default | Effect |
|---|---|---|
| `LoafDistanceMin` | `2` | Nearest distance (m) a stopped pet settles from you instead of standing on you. |
| `LoafDistanceMax` | `4` | Farthest settle distance. `0` = loafing off (settles on you). |
| `LoafDistanceRepick` | `3` | How far you must move before the pet picks a new loaf spot (stops it orbiting you). |

## [Systems] — simulation, loyalty, hunger, stats

| Key | Default | Effect |
|---|---|---|
| `InitialLoyalty` | `50` | Loyalty a freshly tamed pet starts with (0–100). |
| `HungerSecondsPerDay` | `1200` | Game-seconds without feeding per "day" of loyalty decay. `0` or negative disables hunger and decay entirely (a footgun). |
| `TempEscalateSeconds` | `30` | Sustained out-of-band seconds to worsen one comfort stage. |
| `TempRecoverSeconds` | `15` | Sustained in-band seconds to recover one comfort stage. |
| `SpeciesDailyDecay` | `15` | Loyalty lost per day without feeding. |
| `LoyaltyGainPercent` | `5` | Percent of every **positive** loyalty gain (feeding, kill credit, first region crossing) that actually lands — the "slow, hard-won bond" lever. Nothing is lost to rounding: the leftover fraction is banked per pet and saved with it, so twenty preferred meals still deliver the ten loyalty one unscaled meal used to. Losses are **not** scaled by this. `100` = the original fast-bonding rates. |
| `SimTickSeconds` | `2` | How often (game-seconds) the pet systems advance. |
| `CastWatchdogWarnSeconds` | `5` | Log (diagnostic only) when the player's cast has been open this long. |
| `CastWatchdogClearSeconds` | `30` | Force-clear a wedged casting flag after this long. `0` = never. |
| `FeedHealAmount` | `10` | Pet HP healed by a feed when the food's diet entry has no `heal` of its own. |
| `EnableFoodHealthRecovery` | `true` | Feeding also starts a vanilla-style over-time Health Recovery regen (level 1–5 per food; taming chow always level 5). |
| `SatiationFraction` | `0.25` | A pet fed within this fraction of its hunger day is fully satiated and refuses all food. `0` = never refuses. |
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
| `CriticalDrainPerMinute` | `6` | Pet HP drained per minute (percent of max) while Critical. |
| `UneasyDecayMult` | `1.5` | Loyalty-decay multiplier while the pet is Uneasy. |
| `SufferingDecayMult` | `2` | Loyalty-decay multiplier while Suffering or Critical. |

See [Temperature & blankets](./temperature-and-blankets.md).

## [Combat] — the pet fighting, damage scaling, vocals

| Key | Default | Effect |
|---|---|---|
| `EnableCombat` | `true` | Let the pet attack what you're fighting. |
| `AttackDamage` | `25` | Base damage per pet hit (used when not using species stats). |
| `AttackInterval` | `1.4` | Seconds between pet attacks. |
| `AggroRange` | `12` | How far the pet will engage an enemy you're fighting. |
| `AttackRange` | `2.6` | Melee reach for a pet attack. |
| `CombatLeashDistance` | `40` | Leash while actively fighting, so the pet can reach enemies engaged at range. |
| `EnableSpecialAttack` | `true` | Allow firing the pet's Hunt as One signature attack. Requires `EnableCombat`. |
| `EngageRange` | `20` | Command Pet Engage: scan range for an enemy in front of you when you have no locked target. |
| `EngageConeDegrees` | `120` | Width of the "in front of you" cone the Engage scan uses. |
| `DisengageRunHomeSeconds` | `12` | After a fight ends far away, keep the relaxed leash this long so the pet runs home instead of teleporting. |
| `PetAttackVocals` | `true` | Play the species attack sound from the body when its bite animation fires. |
| `PetDamageScalePercent` | `100` | Global balance lever: scale every hit the pet deals, as a percent. `100` = unchanged. Covers all four pet damage paths. |
| `SpeciesDamageScales` | *(empty)* | Per-species overrides of the above, as `Name=percent` pairs (e.g. `Hyena=60`). Exact name wins, else longest match. |
| `UngovernedPetDamagePercent` | `50` | Pet damage while its owner has **not** learned Wild Unknown. `100` = no penalty. |

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
| `EnableRangedSpecial` | `true` | A `Kind=Ranged` species (e.g. Mantis Shrimp) fires its own captured projectile on Hunt as One. |
| `BoltMaxFlightSeconds` | `6` | Backstop ceiling on waiting for a fired bolt's impact before ruling a miss. |
| `SyncSkillCooldownToPet` | `true` | Make the skill advertise the active species' real signature cooldown, so the hotbar countdown is truthful. |
| `BaseSkillCooldownSeconds` | `1` | The skill's own debounce cooldown when there's no species cooldown to sync to. |

## [Synergy] — the Hunt as One stacking buff

| Key | Default | Effect |
|---|---|---|
| `EnableSynergy` | `true` | Landing both halves of Hunt as One grants a stack of Synergy (+all-damage for player and pet). |
| `PercentPerStack` | `0.5` | All-damage bonus percent per stack. |
| `MaxStacks` | `4` | Stack cap. |
| `DurationSeconds` | `60` | Lifespan of the stacks (a new grant refreshes all). |
| `WindowSeconds` | `6` | How long after a cast the two halves may land and still count. |
| `RequireSameTarget` | `false` | Require the player's hit to be on the pet's target to count. |
| `PlayerMeleeRangeMeters` | `3.5` | How close you must be to the pet's target for the synced melee strike to connect. |
| `PlayerMeleeConeDegrees` | `100` | Front-cone width the target must be inside for the synced strike to connect. |
| `PetMeleeRangeMeters` | `4.5` | How close the pet must be to its target for its melee special to connect. |
| `ObserverPatch` | `false` | Diagnostic only (not a gate): log the player's real weapon connects while a Synergy attempt is open, to compare against the judged verdict. Decided at startup — relaunch to change. |

## [ForTheKill] — the Synergy spender

| Key | Default | Effect |
|---|---|---|
| `EnableForTheKill` | `true` | Consume all Synergy stacks for one joint execute strike plus the species' debuff. |
| `CooldownSeconds` | `60` | For the Kill cooldown. Every cast burns it. |
| `BaseDamageMult` | `1.5` | Base multiplier on the joint strike before per-stack scaling. |
| `DamagePerStackPercent` | `35` | Extra damage percent per stack consumed. |
| `EnableKillFavor` | `true` | A kill by the execute grants the player a loyalty-tiered stat buff (the species' `killBuff`). |
| `KillFavorDurationSeconds` | `300` | How long that buff lasts. |

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
| `EnableFeatherFletching` | `true` | Pearlbird feathers enchant an arrow stack in place with quality-tiered "Pearl Fletching". |
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

## Single-switch systems

| Section · Key | Default | Effect |
|---|---|---|
| `[Sigils] EnableSigilSynergies` | `true` | A species with a sigils entry changes its Hunt as One hit while the pet stands in a mage sigil. |
| `[Gear] EnableGearEffects` | `true` | A worn item with a gear entry alters the active pet (e.g. a Pearlbird Mask boosts loyalty gain). |
| `[BuffFoods] EnableBuffFoods` | `true` | Feeding a buff food grants a temporary damage buff. It fills the belly too only if the species' diet also accepts that item (then one item does both); otherwise it is buff-only. |
| `[Scavenge] EnableScavengeBonus` | `true` | A species's listed loot containers roll extra times on first open, by loyalty tier. |
| `[HUD] EnableHealthHud` | `true` | Show a simple pet-health bar (top-left) while a live pet exists. |

## [Skills] — the skill-tree gates

| Key | Default | Effect |
|---|---|---|
| `ScatologyGatesScrolls` | `true` | Recipe scrolls only drop once someone in the session has learned Scatology. |
| `BeastOfBurdenCapacity` | `5` | Flat bag capacity the Beast of Burden passive adds while you have a pet. |
| `EnableWildUnknownGate` | `true` | Region travel breaks the bond unless the owner has learned Wild Unknown (which also grants +5 loyalty per new region pair). |
| `EnableCommunionGate` | `true` | The bond's passive player buffs require the Communion skill. (This default withholds buffs on existing saves until Communion is bought.) |

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
| `AnchorLeashDistance` | `20` | Teleport the anchor to the player when it falls this far behind (out of combat). |
| `AnchorRespawnSeconds` | `60` | How long a downed pet stays gone before re-forming. |
| `AnchorBaseHealth` | `110` | Base for the pet's combat HP (×1.5 at loyalty 100, ×0.5 at loyalty 0). |
| `AnchorDealsDamage` | `false` | Let the anchor's invisible weapon deal damage. Off = the visible-combat system owns damage. |
| `AnchorSpeciesVoice` | `true` | Species-correct hurt/death cries + silencing of the ghost's stock audio. |
| `GlueMode` | `Always` | How the anchor is position-coupled to the visible body. `Always` = welded every frame. |
| `GlueOffsetBehind` | `0.3` | Weld offset (m) behind the body along its facing. |
| `UnifyTargets` | `true` | Keep the anchor and pet locked on the same enemy. |
| `AnchorPlayerCollision` | `PassPlayer` | How the anchor collides with players: `PassPlayer` (can't shove you), `Block` (pre-fix), `Phantom` (blocks nothing). |

## [PetPanel] — Companion-tab controller nav

| Key | Default | Effect |
|---|---|---|
| `EnableControllerNav` | `true` | Right stick scrolls the active half of the Companion tab; triggers switch halves. |
| `StickScrollSpeed` | `1.2` | Scroll speed at full stick deflection. |
| `StickDeadzone` | `0.2` | Right-stick deadzone before scrolling. |

## [Taming] — the player taming loop

| Key | Default | Effect |
|---|---|---|
| `EnableTamingFoods` | `true` | The taming loop: recipe-scroll drops, cooking chow, and taming a wild one by using it nearby. |
| `TameRadius` | `15` | How close a wild creature must be for using its taming food to tame it. |
| `RecipeDropChance` | `1` | Global chance (0–1) a tameable creature drops its recipe scroll on death, ×its per-species chance. |

## [Maren] — the trainer NPC

| Key | Default | Effect |
|---|---|---|
| `EnableMaren` | `true` | Register Maren the trainer with StoryKit. Off = she never appears (relaunch to change). |
| `MarenScene` | `CierzoNewTerrain` | Scene build name Maren stands in. |
| `MarenPosX` / `MarenPosY` / `MarenPosZ` | *(Cierzo spot)* | Maren's spawn position. Re-stamp with the `marenhere` verb. |
| `MarenRotY` | *(Cierzo facing)* | Maren's facing (yaw degrees). |

## [MP] — multiplayer

| Key | Default | Effect |
|---|---|---|
| `AllowConsumeTameInRoom` | `true` | Allow the host's consume-tame while other players are connected. (A known cost: the tamed creature may appear frozen to guests until their next area load.) |

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
> `AlwaysWarmSpecies`) moved to CompanionKit's own config (`cobalt.companionkit.cfg`); the copies here
> are read once to migrate a customised value and then ignored. Edit CompanionKit's config instead.

## See also

- [Beastwhispering overview](./README.md)
- [Dev verbs](./dev-verbs.md) — the `reloadcfg` verb and the command channel
- [Skills](./skills.md) · [Combat & companion](./combat-and-companion.md) · [Temperature & blankets](./temperature-and-blankets.md)
- [Data manifests](./data-manifests.md) — the per-creature data behind these systems
- [Mods index](../README.md)
