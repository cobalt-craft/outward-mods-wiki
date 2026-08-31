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

### In co-op

The "already resident" test above is now a **party** test, not a per-machine one. If the host picks a
species a guest has no body for, that guest has to load a donor scene mid-fight — measured at 3.6 s
to 14.7 s in the field, which is long enough to drop them from the room. So DangerousRoads asks
[SpawnKit](../kits/spawnkit.md) whether the *whole room* can mint a species, not just this box:

- **The roster predicate is room-wide.** A species is a candidate only if it can be minted here *and*
  by every peer that has published a warm set. Unmodded and still-loading players never veto, and
  solo play behaves exactly as before.
- **Prewarming is party-aware.** The region's warm order puts first the species *the party* lacks but
  the host already holds — one request and they become room-wide spawnable at no cost to the host —
  then the ones nobody holds, then the rest.
- **Arming a region pushes a warm request.** When the ambush director arms for a new region it hands
  SpawnKit its top prewarm targets, so every guest starts warming the same short list and the shared
  roster fills instead of each machine warming a different half of it.
- **When peers cost you species, it says so.** The log names the species that were warm here but cold
  in the room and who was cold, reprinting only when the situation changes, and one on-screen toast
  per region arm tells the player the ambush pool is temporarily smaller. `roadsroster` carries a
  room column: **● room-wide warm · ◐ warm here, cold on a peer · ○ cold here**.

- **A refused encounter says why, once.** The wave, the merchant raid, the far-cache restore, the
  stranger's mobs and the patrol's troglodytes all check the same room-wide answer and refuse *before*
  spawning rather than showing a guest a ghost — so in co-op you will sometimes see fewer encounters
  than you would solo. **That is the gate working, not a bug.** The host log carries one dense line
  naming every species currently being refused and the actor number of each cold peer
  (`co-op encounters refused in '<region>': …`), reprinted only when the situation changes or once a
  minute; `skwarmdump` on the host prints each peer's row. A far-parked creature is *held*, never
  dropped, while a peer is cold.

**How strict the gate is is still SpawnKit's knob; DR adds only the announcement.** The one setting
DangerousRoads owns here is `[Coop] AnnounceQuietRoads` (default `false`), which shows the player one
toast per region — *"The roads are quiet — a companion's world is still warming (N species)"* — when
an encounter is refused for a cold peer. It is host-side only, testers are the audience, and with it
off the host log line above still records the fact. How strict the party gate itself is lives in
SpawnKit's own
`[Coop] RoomWarmMode` (`BepInEx/config/cobalt.spawnkit.cfg`, default `RoomDegraded` — spawn anyway
and ask the guests to warm; `RoomStrict` refuses instead), so one knob governs every consumer of the
spawner rather than the ambush director and the spawn menu disagreeing about the same room. DR's own
`[Species] WarmOnly` keeps its original meaning and still governs only the local half.

*Built 2026-08-20, not live-verified — see `docs/guest-mirror-harvest-testplan.md` (GM10–GM19).*

## The wandering merchant

Not every event is hostile. Sometimes the toast reads *"A merchant is on the road to the east"*: a
**Travelling Merchant** with a trade backpack has appeared out of sight and is walking the road to
one of this region's exits, where he leaves the map. Catch him and talk: you can open his shop
(stocked like the region's roaming caravan trader) or let him go. He is
friendly, hard to kill and nearly harmless — and if something attacks him and you drive it off, he
says so the next time you talk. He is never saved and carries no loot.

**He stops for you.** Walk up to him and he halts and turns to face you so you can talk; once you
step away he waits a few seconds and walks on (`[Merchant] GreetRadius`, `GreetResumeSeconds`).

**You cannot hurt him.** Your weapons, lock-on, your pet and any ally on your side ignore him
(`[Merchant] Protected`) — so a pet that picks fights will not drag him into one. Bandits can.

**Bandits.** Most merchants (`RaidChance`, default 75%) get raided: as you come within
`RaidTriggerMeters` of him — or after `RaidDelaySeconds` if you never do — two or three bandits from
this region appear nearby and go for him ("Bandits are closing on the merchant!"). He is tough
enough to last until you get there; drive them off and his greeting changes.

```ini
[Merchant]
WalkSpeed = 0.5
GreetRadius = 4
GreetResumeSeconds = 3
Protected = true
RaidChance = 0.75
RaidMin = 2
RaidMax = 3
RaidTriggerMeters = 90
RaidDelaySeconds = 120
```

`[Events] MerchantWeight` is how often he comes up relative to an ambush (`0` = never);
`MerchantCooldownSeconds` is the shortest gap between two merchants. Only one is on the road at a time.

## The town-guard patrol

Another peaceful-ish card: *"A town patrol is fighting troglodytes to the east."* Two or three
guards in the nearest town's colours — Cierzo plate, Berg's wolf plate, Levant's elite plate,
Monsoon silver, Harmattan's antique plate, New Sirocco militia — are holding the road against four
to six troglodytes, and they are genuinely losing people. You can walk past.

If you **help** — any damage to any of the trogs, yours or your pet's or an arrow's — the guard who
is left standing offers you the town's bounty afterwards: five or six silver for each troglodyte
that fell, paid once. If you only watched, you get thanks and a warning. Either way they then form
up and walk off toward the far end of the road, and are gone once you can no longer see them.

You cannot hurt the guards (`[Patrol] GuardProtected`), so a pet that picks fights will not turn the
rescue into a brawl. The troglodytes can, and do.

```ini
[Patrol]
GuardMin = 2
GuardMax = 3
TrogMin = 4
TrogMax = 6
GuardHealth = 450
GuardProtection = 12
GuardDamageMult = 0.9
GuardProtected = true
RewardSilverMin = 5
RewardSilverMax = 6
StalemateSeconds = 90
LingerSeconds = 20
RemoveRadiusMeters = 120
```

`[Events] PatrolWeight` is how often a patrol comes up relative to an ambush (`0` = never);
`PatrolCooldownSeconds` is the shortest gap between two of them. One patrol is on the road at a time,
and the troglodytes count against the same `[Wave] MaxOwnActive` cap as any other spawn — which is
why `RemoveRadiusMeters` matters: the guards give their slots back only once nobody can see them.

*Built 2026-08-26, not live-verified — see `docs/road-patrol-testplan.md` (PT0–PT21).*

## The familiar stranger

*"A traveller is under attack to the east."* One or two of **your other saved characters** — the
ones on this machine's character-select screen, built into look-alikes from their saves (face, hair,
armour, weapon, backpack) — are on the road fighting two to four of the region's own creatures. They
are not invincible. Walk past and they may die.

Save them (any damage from you, your pet or an arrow, and every mob dead with at least one of them
standing) and the survivor says: *"Thank you stranger. You seem familiar. I don't have much to
offer but my life is worth more than any of the trinkets in my backpack. If you see anything you like
it is yours."* Pick **Look in the backpack** and that character's real backpack and pouch open in the
ordinary container window. Take **one** item or stack: it goes into your inventory, the window
closes, and it is **permanently removed from the other character's save** (written as a new save
snapshot — the next time you play that character, the item is gone). One pick per stranger per
encounter; afterwards they just say *"Take care, stranger."*

The character you are currently playing is never one of them, nor is a retired one. With no other
saved character on the machine the card simply never comes up (one warning in the log at boot).

```ini
[Events]
StrangerWeight = 0.25
StrangerCooldownSeconds = 900

[Stranger]
MobsMin = 2
MobsMax = 4
MaxStrangers = 2
StalemateSeconds = 90
RewardEnabled = true
```

`RewardEnabled = false` keeps the rescue line but never opens the backpack and never touches any
save. Commands: `roadsstranger [status|saves|spawn|resolve|help|despawn]`, `roadscard stranger
[at=here count=<1|2> mobs=<n>]`. Log tag `[STRANGER]` (the save write logs `[SAVEHANDOFF]`).

*Built 2026-08-29, not live-verified — see `docs/stranger-testplan.md` (ST0–ST10). Single-player
first; in co-op the card is host-only and a guest clicking the row is refused.*

**Finding him.** Besides the toast's bearing, the HUD compass shows a **green dot** for the merchant
from the moment he appears until he leaves — at any distance, slightly larger than the red hostile
blips, so it reads as a place to walk toward rather than a threat. `[Compass] ShowMerchantBlip`
turns it off and `MerchantBlipColor` recolours it (an unparseable colour falls back to green, never
to hostile red):

```ini
[Compass]
ShowMerchantBlip = true
MerchantBlipColor = #34C759
```

In co-op you may see another compass marker: that one is the base game's own other-player element,
pointing at your partner, and is not drawn by DangerousRoads.

If the road is blocked on the way — a roadside point the navmesh cannot reach — he skips past it
and keeps walking rather than standing there; only a stall at the very gate, or several in a row,
ends the walk early (he despawns, and the log says why).

## The Levant shakedown ("Slumbush")

The first card that is dealt **in a city**, and the first with a rule of its own: it only ever
joins the deck in **Levant**. About 3% of city draws there (the rest are blanks — see below): a
shaking voice behind you says *"Don't turn around. Leave 200 silver on the ground and you get to
walk away."* You cannot move while the voice speaks (the game's own bed-lock, released the moment
the box closes; while it is up, advance the text with the mouse — the halt also swallows the
"continue" key). Two answers:

- *"This place seems like trouble, I better not start any."* — 200 silver leaves your inventory
  (bag first, then pouch) and the thugs melt away. If you cannot cover it, they attack anyway.
- *"Thugs like this are worse than the scourge."* — 1–3 thugs in tattered clothes with iron
  weapons, standing ~7 m behind you, turn hostile. One of them carries 60 silver; loot his corpse.
  Closing the box without answering counts as refusing.

Log tag `[SLUM]`. Force it anywhere with `roadscard slumbush [count=1..3 demand=<silver> loot=<silver> radius=<m>]`
(the log notes when you forced it somewhere it would have sat out). Tuning lives in `[Slumbush]`
(`DemandSilver`, `LootSilver`, `SpawnRadius`, `DecisionSeconds`, `LeaveSeconds`, `StalemateSeconds`,
`ReplySeconds`, `ThugHealth`, `ThugDamageMult`, `ThugChanceToAttack`) and `[Events] SlumbushWeight / SlumbushCooldownSeconds / SlumbushWhen`.

## Decks per kind of place, and the blank card

Every mapped area is now dealt a deck — the **overworld** deck (ambush, merchant, patrol,
stranger), a **city** deck and a **dungeon** deck. The road cards carry `zone:overworld` in their
own rule, so they never turn up in town or underground. What makes the other decks work is the
**blank card**: an event that does nothing, counts as fired, and re-arms the clock. Its weight is
the one per-zone number in the deck:

```ini
[Events]
BlankWeightOverworld = 0      # the road deck exactly as before; raise it to thin out road events
BlankWeightCity = 0.97        # + SlumbushWeight 0.03 → ~3% shakedowns in Levant, all blanks elsewhere
BlankWeightDungeon = 1.0      # no dungeon card yet: every dungeon draw is a blank
```

`[Events] DeckEnabled = false` still means "the road mod exactly as before": ambushes on the road,
and nothing at all in a city or dungeon.

`roadsdeck` shows the deck for the zone you are standing in (`world zone=City …`), with the road
cards listed as `SITS OUT — needs zone:overworld`.

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

### Roads feel too persistent

A creature that is **fighting** is never parked by the far cache, however far you run — outrunning an
ambush no longer ends it, and it keeps holding one of your `[Wave] MaxOwnActive` slots until it gives
up and disengages. That is deliberate: a pursuer vanishing mid-swing read as a bug. It is also a
recent change (2026-08-30). Before it, the exemption was asking the wrong thing about an AI body and
so **never once fired** — every far fight was parked mid-swing, which is why a town patrol could
report itself resolved with nobody alive and nobody dead.

If you preferred the old, quieter roads, `[FarCache] ParkMidFightBeyond` gives a bounded version of
it back: a hard distance past which even an active fight is parked. `0` (the default) means never.

```ini
[FarCache]
## The one way to park a creature that is FIGHTING: a hard distance (meters, 3D) past which even an
## active fight is parked. 0 = never.
# Setting type: Single
# Default value: 0
ParkMidFightBeyond = 300
```

Live-tunable — edit `BepInEx/config/cobalt.dangerousroads.cfg` and run `reloadcfg`. A value below
`[FarCache] DespawnRadius` is raised to it (a fighting creature is never parked closer than a
peaceful one); `roadscache` and the mod's config summary line both report the effective figure.

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
| `[Events]` | `DeckEnabled` | `true` | Draw each event from the deck (ambush or merchant). `false` = every event is an ambush, exactly as before the deck. |
| | `AmbushWeight` | `1.0` | Relative draw weight of the hostile wave. |
| | `MerchantWeight` | `0.35` | Relative draw weight of the wandering merchant (`0` = never). |
| | `MerchantCooldownSeconds` | `600` | Shortest gap between two merchants, stamped when drawn. |
| | `BlankWeightOverworld` / `BlankWeightCity` / `BlankWeightDungeon` | `0` / `0.97` / `1.0` | Weight of the do-nothing blank card in each kind of place (see [Decks per kind of place](#decks-per-kind-of-place-and-the-blank-card)). |
| | `SlumbushWeight` / `SlumbushCooldownSeconds` | `0.03` / `1800` | The Levant shakedown card's weight (city deck only) and cooldown. |
| | `AmbushWhen` / `MerchantWhen` / `PatrolWhen` / `StrangerWhen` / `SlumbushWhen` | *(empty)* | When that card may **join the deck** — a world rule checked before every draw (see [When a card may join the deck](#when-a-card-may-join-the-deck)). Empty = always. |
| `[Merchant]` | `Health` | `900` | The merchant's max health. Applies to the next merchant. |
| | `Protection` | `20` | Flat damage protection. Next merchant. |
| | `DamageMult` | `0.25` | Multiplier on the damage he deals (`1` = unchanged). Next merchant. |
| | `WalkSpeed` | `0.5` | His walking speed: a multiplier on the body's run speed (vanilla wanderers 0.3, `1.1` is a jog). Live. |
| | `ExitRadius` | `6` | How close to the exit counts as arrived (he despawns there). Live. |
| | `StuckSeconds` | `90` | Despawn after this long with no progress outside combat (`0` = never). Live. |
| | `DespawnDeferSeconds` | `180` | He never leaves while his shop panel or his dialogue is open — arriving at the exit and giving up as stuck both **wait** (he stands still) until it closes, because destroying him mid-trade leaves the shopper's UI stuck on a transaction that can never finish. This is the ceiling on that wait: past it he goes anyway with a warning. `0` or less = wait for as long as the shop is open. Live. |
| | `MinRouteMeters` | `150` | Prefer an exit at least this far from where he appears. Next merchant. |
| | `DefendsHimself` | `true` | Fights back against bandits (never you). `false` = fully passive. Next merchant. |
| | `GreetRadius` | `4` | Stops and faces a player this close (metres); `0` = never stops (dialogue still holds him). Live. |
| | `GreetResumeSeconds` | `3` | Walks on this long after the last player leaves. Live. |
| | `Protected` | `true` | The player side (weapons, lock-on, pets, allies) cannot hurt or target him; bandits can. Next merchant. |
| | `RaidChance` | `0.75` | Chance (0–1) that bandits raid him during the walk. Next merchant. |
| | `RaidMin` / `RaidMax` | `2` / `3` | How many raiders; each counts against `[Wave] MaxOwnActive`. Next merchant. |
| | `RaidTriggerMeters` | `90` | The raid springs when a player is this close (`0` = by delay only). Live. |
| | `RaidDelaySeconds` | `120` | The raid springs this long after his spawn regardless (`0` = by proximity only). Live. |
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
| `[Coop]` | `AnnounceQuietRoads` | `false` | Host-only. Show the player one toast per region when a co-op encounter is refused because a peer cannot show the species yet ("The roads are quiet — a companion's world is still warming"). Off by default: the host log line names the species and the cold peers either way. How strict the gate itself is lives in SpawnKit's `[Coop] RoomWarmMode`. |
| `[FarCache]` | `Enabled` | `true` | Park a creature nobody is near (silent despawn, position/facing/health remembered) and put it back when a player returns, so an abandoned wave stops holding a `[Wave] MaxOwnActive` slot for the rest of the zone visit. |
| | `DespawnRadius` | `120` | Park when **every** player is farther than this (metres). |
| | `RestoreRadius` | `80` | Put it back when any player comes within this of the parked spot. Clamped in code to at most 0.8 × `DespawnRadius` (the hysteresis that stops park/restore flapping). |
| | `MaxEntries` | `24` | Ledger cap; past it the oldest parked creature is forgotten. |
| | `MaxAgeMinutes` | `20` | Forget a parked creature after this long (`0` = never). |
| | `ParkMidFightBeyond` | `0` | The only way to park a creature that is **fighting**: a hard distance past which even an active fight parks. `0` = never. See [Roads feel too persistent](#roads-feel-too-persistent) — this knob exists because until 2026-08-30 the "never park a fight" rule silently never worked. Values below `DespawnRadius` are raised to it. |
| `[Notify]` | `ShowToast` | `true` | On-screen message when a wave arrives. |
| | `ToastMinGapSeconds` | `30` | Minimum gap between toasts. |
| | `ToastBearing` | `true` | Name the compass direction in the toast. |
| `[Compass]` | `ShowBlips` | `true` | Mark this mod's live spawns on the HUD compass. |
| | `BlipRangeMeters` | `250` | Stop marking a creature past this distance. |
| | `BlipColor` | `#FF3B30` | Blip colour, `#RRGGBB` or `#RRGGBBAA`. |
| | `ShowMerchantBlip` | `true` | The wandering merchant's green compass dot, any distance, independent of `ShowBlips`. Live. |
| | `MerchantBlipColor` | `#34C759` | The merchant dot's colour; drawn 1.4× a hostile blip. Unparseable = green. Live. |
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

[Events]
DeckEnabled = true
AmbushWeight = 1.0
MerchantWeight = 0.35
MerchantCooldownSeconds = 600
# Merchants keep to daylight; the stranger only turns up once you've joined a faction.
MerchantWhen = day
StrangerWhen = faction:BlueChamber | faction:HeroicKingdom | faction:HolyMission

[Merchant]
Health = 900
WalkSpeed = 0.5
StuckSeconds = 90
# Never vanish out from under an open shop panel; give up waiting after three minutes.
DespawnDeferSeconds = 180
RaidChance = 0.75

[Placement]
MinDistanceMeters = 40
MaxDistanceMeters = 200

[Species]
ExtraBlocklist = Wolfgang Veteran,Wolfgang Mercenary

[Coop]
# Testers: announce a co-op encounter refused because a guest is still warming (one toast per region).
AnnounceQuietRoads = false

[FarCache]
DespawnRadius = 120
RestoreRadius = 80
# 0 = a creature that is fighting is never parked, however far you run. Set a distance (try 2-3x
# DespawnRadius) if you would rather a fight you have left behind quietly ends.
ParkMidFightBeyond = 0

[Notify]
ShowToast = true

[Compass]
ShowBlips = true
ShowMerchantBlip = true
MerchantBlipColor = #34C759
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
| `roadscard slumbush [count= demand= loot= radius=]` | Force the Levant shakedown here and now (any zone). |
| `roadsdeck [draw\|<id> [key=value…]]` | The event deck: the world it was last built for, which cards joined (`ok`) or sat out (`SITS OUT — needs …`), live weights, cooldowns, last draw. `draw` forces a draw; `<id>` (`ambush`, `merchant`, …) forces that card even if it would have sat out (the log says so), with the same optional keys as `roadscard`. |
| `roadscard [id] [key=value…]` | Force one card with optional overrides; every key is optional, so `roadscard merchant` is a plain draw of the merchant and `roadscard` alone is a real deck draw. Merchant: `exit=<n from roadsexits>` `dest=<exit-name substring>` `health=` `protection=` `damage=<mult>` `speed=` `defends=true\|false` `at=here` (spawn in front of you) `name=` `raid=0\|1` (forbid/force the bandit raid) `raiders=<n>` `raidspecies=<key>` `outfit=<n>` (pin an outfit from the pool) `protected=true\|false`. Ambush: `count=` `species=`. An unknown key is warned and ignored; an `exit`/`dest` that names nothing is refused. |
| `roadsmerchant [status\|spawn [key=value…]\|despawn\|rescue\|raid\|goto <exit>]` | The wandering merchant: where he is (walk, greet pause, raid state) / put one on the road now (same keys as `roadscard merchant`) / remove him / fake the rescue so the thanks greeting shows / spring the bandit raid now / re-route him to an exit index from `roadsexits`. |
| `roadsexits` | Census of this scene's zone exits: index, destination, distance and bearing from you. |
| `roadsstock` | Which live merchant in the scene carries the caravan stock table the road merchant copies. |
| `roadsarm` | Arm the director so the next wave is due immediately. |
| `roadsdisarm` | Stop the ambush timer until the next region change. |
| `roadswarm <species>` | Queue a species for background prewarm now. |
| `roadssweep [kill]` | Despawn every creature this mod spawned. `kill` kills them instead (death animation and loot). The panic button. |
| `roadsui [depth]` | Dump the live HUD hierarchy and report whether a compass was found. |
| `roadsblips` | Compass blip status: compass found, HUDs hooked, blips live (hostile and merchant counted separately). |
| `selftest` | Run the built-in checks and log a pass/fail report. |

The shared [ForgeKit](../kits/forgekit.md) command pack answers on this channel too, so `reloadcfg`,
`teleport`, `pos`, `settime`, `scenedump` and the rest are available here as well.

## When a card may join the deck

Drawing a card is two steps. First the mod takes one snapshot of the world — game hour, night or
day, rain, the region you are in, your story faction (Blue Chamber / Heroic Kingdom / Holy Mission
/ none) and the quest flags you hold — and asks every card whether it wants to be in the deck
*right now*. Only the cards that said yes are shuffled; the rest sit out, take no share of the roll
and burn no cooldown. A card that sits out can leave the ambush alone in the deck, in which case
no roll is pulled at all — a night-time deck with a daytime-only merchant is byte-identical to the
mod before the deck existed.

Each card's rule is the `[Events] <Card>When` line in
`BepInEx/config/cobalt.dangerousroads.cfg`. Empty means always. The grammar is small enough to fit
on one line:

| Term | Holds when |
|---|---|
| `always` | always (the same as leaving the line empty) |
| `night` / `day` | the game says it is night (22:00–05:00 in vanilla) / is not |
| `hour=20-6` | the game clock is in the window — start inclusive, end exclusive, wraps midnight |
| `rain>0.5` / `rain<0.2` | rain intensity (0–1) at least / below the number |
| `flag:<quest event UID>` | you hold that quest event (story progression) |
| `faction:<name>` | your story faction contains the text: `BlueChamber`, `HeroicKingdom`, `HolyMission`, `None` |
| `region:<text>` | the current area's name contains the text |
| `zone:overworld` / `zone:city` / `zone:dungeon` | the kind of place you are in (the game's town/city flag; every other mapped non-overworld area counts as a dungeon) |

Join terms with `&` (all must hold) and `|` (any may hold); `!` in front of a term negates it.
`&` binds looser than `|`, so `night & rain>0.3 | flag:X` reads as *night, and either rain or the
flag*. Whitespace is free.

```ini
[Events]
MerchantWhen = day & rain<0.5
PatrolWhen = !night | faction:HeroicKingdom
StrangerWhen = flag:SomeQuestEventUID & region:Chersonese
```

A line that does not parse is logged once (`[EVENT] [Events] merchantWhen '…' does not parse: …`)
and the card is treated as **always** eligible — a typo never makes a card silently vanish.
`roadsdeck` prints the world the deck was last built for and, per card, the rule and why it sat
out; `reloadcfg` picks up an edited rule without a relaunch.

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

**The event deck is built in two objects, on purpose.** `WorldSnapshot` (an immutable value the
director captures once per draw through `WorldProbe`) is handed to every card's `ICardCondition`
(Specification pattern — leaves like `HourWindow`, `QuestFlagCondition`; composites `AllOf` /
`AnyOf` / `Not`; `Always` as the null object; `CardCondition.Parse` is the config interpreter).
`EligibleDeck.Build(cards, world)` is the builder that asks each card and keeps the refusals with
their reason; `EventDeck.Draw` then rolls over what joined and knows nothing about the world. A card
gets a code-level rule through `IEventCard.Condition` (ANDed with the operator's `When` text), so a
story card can insist on its own plot flag without a config line. All of it except `WorldProbe` is
in `core/DangerousRoads.Core/` and unit-tested.

## See also

- [Mods index](README.md)
- [Installing](../installing.md)
- [SpawnKit](../kits/spawnkit.md) — the spawning engine underneath
- [DonorKit](../kits/donorkit.md) — where the creature bodies come from
- [Beastwhispering](beastwhispering/README.md) — works alongside it
