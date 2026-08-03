# DangerousRoads — the wilds are not empty anymore

**DangerousRoads** is a mod for Outward that repopulates the overworld with wandering hostile
creatures, so a stretch of country you have already cleared is country you have to watch again.

**At a glance**
- Type: gameplay mod
- Requires: BepInEx 5 (Mono branch) · [ForgeKit](../kits/forgekit.md), [SpawnKit](../kits/spawnkit.md)
  and [CompanionKit](../kits/companionkit.md) directly — and, underneath those,
  [DonorKit](../kits/donorkit.md), [AggroKit](../kits/aggrokit.md) and [NetKit](../kits/netkit.md)
- Config: `BepInEx/config/cobalt.dangerousroads.cfg`
- Commands: `BepInEx/config/DangerousRoads_cmd.txt`

> **Alpha, and single-player only for now.** Co-op is unsupported and there is a known problem with
> one creature family — see [Known issues](#known-issues). Back up saves you care about.

## For players

Aurai's overworld is hand-placed and finite. Once you have cleared the road between two points it
stays cleared, and the walk back is a walk through an empty map.

DangerousRoads fills it back in. While you are travelling out in the wilds, hostile creatures can
turn up anywhere — out of sight, off the road, behind you. They are ordinary wild creatures native
to the region you are in: Enmerkar sends beast golems and bandits, Abrassar sends bugs and bandits.
They wander, they notice you, they fight, and they drop their normal loot.

There is nothing to switch on and no key to press. Install it, walk out of town, and give it a
minute.

**What it deliberately does not do**

- **Never in towns or villages.** Settlements stay safe.
- **Never in dungeons, caves or interiors.** Only out in the open.
- **Never touches your save.** Its creatures are temporary — they are gone when you leave the area,
  and nothing about them is written to your save file. Uninstalling is deleting the folder.
- **No bosses, no story characters, no custom content.** Everything it puts in the world is vanilla.

## How it works

- A **region watcher** notices which area you are in and arms an ambush timer. Only the six
  open-world regions — Chersonese, Hallowed Marsh, Abrassar, Antique Plateau, Enmerkar Forest and
  Caldera — are on the list, and it fails closed: an area the mod does not recognise (a DLC area, a
  modded scene, a menu, mid-transition) is never treated as overworld. The game's own town/city
  classifier can also veto an area on the list, so settlements stay safe even if a future patch
  renumbers something.
- The timer fires on a randomised interval measured in seconds of *actual play* — it freezes while
  the game is paused or loading, so an ambush cannot bank up while you sit in the inventory.
- A **placement pass** looks for a spot 40–200 m away that the player cannot see, that is genuinely
  walkable from where you stand, and that is flat ground rather than a ledge or a roof. It tries
  several anchor sources in order (live AI, gathering spots, the game's own wandering-squad spawn
  points, world interactables, then a procedural fallback) and gives up quietly if none of them
  produce a verified spot.
- A **wave** of 1–3 creatures is then spawned through [SpawnKit](../kits/spawnkit.md) as real
  vanilla hostiles. Every member of a wave is the same species — that is structural, not a setting:
  same species means the same faction, which is what stops an "ambush" from fighting itself.
- Creatures are only drawn from species whose body is already loaded and resident. A small
  background prewarm keeps a few of the region's species ready. This is what keeps an ambush from
  costing you a stall mid-play.
- Arrivals are announced by an on-screen toast naming the compass direction, and marked on the HUD
  compass while they are in range.

## Known issues

- **A Wolfgang can turn up on the wrong side.** The Wolfgang family's allegiance is driven by your
  plot and relationship progression in the base game, not by a fixed table value — so a wave that
  draws one can spawn it in an allegiance the mod did not intend, including friendly. The fix is to
  drop the whole family from the encounter roster (hard-coding a faction for them would only be
  wrong at a different point in the story), and that is **not in 0.1.0**. To stop it now, add the
  names to `[Species] ExtraBlocklist` and relaunch: `Wolfgang Captain` is blocked by default,
  `Wolfgang Veteran` and `Wolfgang Mercenary` are not.
- **A few other species report an unstable faction.** The Mantis family (Rock Mantis, Mana Mantis,
  Mantis Shrimp), Veaber, and Caldera's Boreo have each been seen reporting different factions on
  different spawns. Under investigation; the symptom is the same as above — an encounter that does
  not fight you.
- **Multiplayer is unsupported.** Only the host ever runs encounters, which is the right shape, but
  no real two-machine session has been played. Treat co-op as untested.
- **Encounter frequency varies by region.** Some parts of the map offer far more good places to put
  a creature than others, so the wilds are livelier in some regions than in others.

### Ambushes stop at a number I didn't set

**Two caps apply, in two different files, and the one you probably found is the one that isn't
stopping you.** Dangerous Roads checks its own `[Wave] MaxOwnActive` *before* it ever asks SpawnKit
to spawn, so raising SpawnKit's `[Spawner] MaxActiveSpawns` on its own changes nothing.

| Cap | File | Scope |
|---|---|---|
| `[Wave] MaxOwnActive` | `BepInEx/config/cobalt.dangerousroads.cfg` | Just this mod's creatures. **Checked first — usually the one stopping you.** |
| `[Spawner] MaxActiveSpawns` | `BepInEx/config/cobalt.spawnkit.cfg` | Every mod that spawns through SpawnKit, added together. |

Raise `MaxOwnActive`, and keep it at or below `MaxActiveSpawns` — going over just means SpawnKit
refuses the spawn and toasts you on every attempt:

```ini
[Wave]
## Cap on creatures this mod may have alive at once.
# Setting type: Int32
# Default value: 8
MaxOwnActive = 12
```

```ini
[Spawner]
## Refuse spawns that would push the live count (incl. pending) past this cap.
# Setting type: Int32
# Default value: 8
MaxActiveSpawns = 12
```

Both are live-tunable — edit and run `reloadcfg`, no relaunch. When the cap is what's holding
waves back, the log says so and names both keys:

```
[AMBUSH] soft block: own cap reached — 8/8 alive. To raise it, edit [Wave] MaxOwnActive in
cobalt.dangerousroads.cfg — NOT [Spawner] MaxActiveSpawns (=8) in cobalt.spawnkit.cfg, which
this check never consults. Retrying in 15s.
```

`roadsstatus` reports the same thing as `active N / M`. Note that **`MaxOwnActive` defaulted to 6
before 0.1.2**, below SpawnKit's default of 8 — and BepInEx never rewrites a value that already
exists in your config file, so an install created before then still has `6` and must be edited by
hand to pick up the new default.

## Settings

All settings live in **`BepInEx/config/cobalt.dangerousroads.cfg`**, created on first launch. Every
key is live-tunable without a relaunch: edit the file and run the `reloadcfg` command (BepInEx 5 has
no config file-watcher of its own, so an edit alone does nothing until then).

| Section | Key | Default | Effect |
|---|---|---|---|
| `[General]` | `Enabled` | `true` | Master switch. `false` + `reloadcfg` disarms instantly. |
| `[Schedule]` | `MinIntervalSeconds` | `20` | Shortest gap between ambushes. |
| | `MaxIntervalSeconds` | `60` | Longest gap. Raise both for a quieter world. |
| | `FirstArmDelaySeconds` | `60` | Grace period after entering a region before the first ambush can fire. |
| | `CooldownAfterWaveSeconds` | `45` | Hard floor after a wave places, whatever the interval rolls. |
| | `RetrySeconds` | `15` | Re-check delay after a blocked attempt (loading, in combat, cap reached). |
| `[Wave]` | `MinCount` | `1` | Fewest creatures in a wave. |
| | `MaxCount` | `3` | Most creatures in a wave. |
| | `MaxOwnActive` | `8` | Cap on creatures this mod may have alive at once. **If ambushes stop below SpawnKit's cap, this is the key to raise** — see [Ambushes stop at a number I didn't set](#ambushes-stop-at-a-number-i-didnt-set). |
| | `MemberSpacingMeters` | `4` | Minimum spacing between members of one wave. |
| | `SkipWhileInCombat` | `true` | Hold a wave back while you are already fighting. |
| | `CombatDeferMaxSeconds` | `120` | Longest continuous stretch that hold may last before the wave fires anyway. `0` = unbounded, which can silence the mod entirely. |
| | `ClusterRadiusMeters` | `12` | How tightly a wave's members group around the lead spot. |
| `[Placement]` | `MinDistanceMeters` | `40` | Closest a creature may appear. |
| | `MaxDistanceMeters` | `200` | Farthest a creature may appear. |
| | `PathLengthRatioMax` | `1.8` | Reject a spot whose walking distance exceeds this multiple of its straight-line distance. |
| | `RequireOutOfSight` | `true` | Only place where geometry blocks your view. |
| | `SightCheckAllPlayers` | `true` | In co-op, require the spot to be hidden from every player. |
| | `SourceOrder` | `liveai,gatherpoints,squadpoints,interactables,procedural` | Anchor sources to try, in order. |
| | `EnableInteractableAnchors` | `true` | Use world interactables (gatherables, chests) as anchors. |
| | `MaxCandidatesPerWave` | `24` | Ceiling on candidate spots verified per wave (a frame-cost guard). |
| | `PlateauProbeRadius` | `1.0` | Half-width of the flat-ground check. Bigger = stricter. |
| `[Species]` | `WarmOnly` | `true` | Only spawn species already resident. Turning this off makes ambushes hitch. |
| | `PrewarmCount` | `3` | How many of the region's species to warm in the background. |
| | `PrewarmIntervalSeconds` | `20` | Gap between background prewarms. |
| | `ExtraBlocklist` | *(empty)* | Comma-separated species never to spawn, appended to the built-in list of named story NPCs and set-piece bosses. Case-insensitive substring match. |
| `[Notify]` | `ShowToast` | `true` | On-screen message when a wave arrives. |
| | `ToastMinGapSeconds` | `30` | Minimum gap between toasts. |
| | `ToastBearing` | `true` | Name the compass direction in the toast. |
| `[Compass]` | `ShowBlips` | `true` | Mark this mod's live spawns on the HUD compass. |
| | `BlipRangeMeters` | `250` | Stop marking a creature past this distance. |
| | `BlipColor` | `#FF3B30` | Blip colour, `#RRGGBB` or `#RRGGBBAA`. |
| `[Diag]` | `LogVerbose` | `false` | One log line per placement candidate with its full verdict chain. |
| | `LedgerAutoDumpSeconds` | `300` | Auto-print the anchor-source measurement table this often (`0` = never). |
| `[Keys]` | `ForceWaveKey` | *(unbound)* | Force a wave immediately. Ships unbound on purpose — keys are a cross-mod resource. |

### Example configuration

```ini
[General]
Enabled = true

[Schedule]
MinIntervalSeconds = 20
MaxIntervalSeconds = 60
FirstArmDelaySeconds = 60

[Wave]
MinCount = 1
MaxCount = 3
SkipWhileInCombat = true

[Placement]
MinDistanceMeters = 40
MaxDistanceMeters = 200

[Species]
ExtraBlocklist = Wolfgang Veteran,Wolfgang Mercenary

[Notify]
ShowToast = true

[Compass]
ShowBlips = true
```

## Commands

Write a verb into **`BepInEx/config/DangerousRoads_cmd.txt`** and it runs on the next poll, even
while the game is paused. Results go to `BepInEx/LogOutput.log`. An unknown verb — or `help` —
lists them all.

| Verb | What it does |
|---|---|
| `roadsstatus` | Director state: region, gate verdict, time to next wave, roster and warm counts, last wave. |
| `roadsroster` | Every species considered for this region: source, faction, blocked, expedition-only, warm. |
| `roadsfactions` | Harvest species→faction from the loaded scene and its reserve squads, then print anything new. Spawns nothing. |
| `roadsledger` | Per-region, per-source candidate counts, accept rates and reject histogram. `roadsledger reset` clears it. |
| `roadsanchors [n]` | Run the whole anchor chain now and print every candidate with its verdict chain. Spawns nothing. |
| `roadssquadpoints` | Raw census of the game's own wandering-squad spawn points in this scene. |
| `roadsprobe [x y z]` | Run the verification pipeline on one point and print every stage. No arguments = where you stand. |
| `roadsmark` | Pick the best anchor right now, log it and draw a marker ray. Spawns nothing. |
| `roadsnow [count] [species…]` | Force a wave immediately, through the real pipeline. |
| `roadsarm` | Arm the director so the next wave is due immediately. |
| `roadsdisarm` | Stop the ambush timer until the next region change. |
| `roadswarm <species>` | Queue a species for background prewarm now. |
| `roadssweep [kill]` | Despawn every creature this mod spawned. `kill` kills them instead (death animation and loot). The panic button. |
| `roadsui [depth]` | Dump the live HUD hierarchy and report whether a compass was found. |
| `roadsblips` | Compass blip status: compass found, HUDs hooked, blips live. |
| `selftest` | Run the built-in checks and log a pass/fail report. |

The shared [ForgeKit](../kits/forgekit.md) command pack answers on this channel too, so `reloadcfg`,
`teleport`, `pos`, `settime`, `scenedump` and the rest are available here as well.

## For modders

DangerousRoads is a thin director on top of [SpawnKit](../kits/spawnkit.md) — it decides *when* and
*where*, and SpawnKit does the spawning. Two pieces are worth knowing if you build something
similar:

- **It does its own placement** and passes an explicit world position to SpawnKit, rather than using
  SpawnKit's spawn-beside-the-player ring, which is a near-field tool.
- **It owns its spawns by tag**, so `roadssweep` cleans up exactly its own creatures and never
  another consumer's. A mod must never call SpawnKit's operator-wide despawn.

Its species roster is derived from [DonorKit](../kits/donorkit.md)'s donor-scene table, the same
roster SpawnKit uses.

## See also

- [Mods index](README.md)
- [Installing](../installing.md)
- [SpawnKit](../kits/spawnkit.md) — the spawning engine underneath
- [DonorKit](../kits/donorkit.md) — where the creature bodies come from
- [Beastwhispering](beastwhispering/README.md) — works alongside it
