# Echoes — your other characters live in the towns

**Echoes** is a small mod for Outward that puts your *other* saved characters into the world as
townsfolk. Whichever region a character last saved in, you will find them strolling that region's
town: saved in Chersonese → Cierzo; Enmerkar → Berg; Abrassar → Levant; Hallowed Marsh → Monsoon;
Antique Plateau → Harmattan; Caldera → New Sirocco. A character who saved *inside* a town wanders
that town.

They look like themselves (face, hair, skin, gender) and wear the armour they had on. They walk the
same routes the town's real NPCs walk. You cannot talk to them, trade with them, hurt them or make
them fight — they are echoes, not companions.

**At a glance**
- Type: gameplay mod (cosmetic)
- Requires: BepInEx 5 (Mono branch), [ForgeKit](../kits/forgekit.md), SideLoader
- Config: `BepInEx/config/cobalt.echoes.cfg`
- Commands: `BepInEx/config/Echoes_cmd.txt`
- Save: nothing — echoes are never saved; they are re-made every time a town loads

## For players

Install, play. Every time you enter a town, each of your other characters whose last save was in
that town's region appears somewhere along a townsperson's route. The character you are currently
playing never echoes. Retired (legacy) characters are skipped unless you turn `IncludeRetired` on.

Save another character somewhere new and their echo moves the next time you load a town (the mod
re-reads your save folders at game start; the `echoreload` command re-reads them without a restart).

### Settings — `BepInEx/config/cobalt.echoes.cfg`

```ini
[General]
## Spawn echoes of your other saved characters in towns.
Enabled = true
## Also echo retired (legacy) characters.
IncludeRetired = false

[Look]
## Give the echo the weapon/shield the character had equipped. Off = armour only.
ShowWeapons = false

[Behaviour]
## Echoes cannot be killed. They never fight back or target anyone regardless.
Invulnerable = true
## Borrow a waypoint patrol when the town has one; otherwise a free-wander area.
PreferPatrols = true
## Walk speed modifier (0.3 = the vanilla stroll, 1.1 = a run). 0 = copy the borrowed NPC's own value.
WanderSpeed = 0
## How many echoes a town shows at once (0 = every eligible character). Which ones is fixed
## per town, so the same faces appear every load.
MaxPerTown = 2

[Multiplayer]
## Spawn echoes while hosting a room with guests. Off by default — a guest has no
## template for the host's saves, so they would see an un-dressed body.
SpawnInMultiplayer = false
```

### Commands — `BepInEx/config/Echoes_cmd.txt`

Write one line into the file and it runs on the next poll.

| Command | What |
|---|---|
| `echoes` | List every saved character, the town it echoes into, whether it belongs to the current scene and is live, and the routes the town offers. |
| `echoreload` | Re-read the save folders and rebuild the echoes (after saving another character). |
| `echospawn [name]` | Put every echo (or one, by name) in the current scene regardless of region — for looking at them. |
| `echodespawn` | Remove every echo in the scene. |

## Notes

- A town shows at most `MaxPerTown` echoes (default 2), always the same ones for that town; raise
  it (or set 0) to see everyone. `echospawn <name>` ignores the cap.
- Echoes exist only in the six towns. Dungeons, the overworld and interiors get none.
- If a town has no walking NPCs at all (an unusual state), the echo stands where it spawned.
- Multiplayer: the host places echoes; guests see them only when `SpawnInMultiplayer` is on,
  and then without the outfit and look (SideLoader needs the template on the guest's side too).
