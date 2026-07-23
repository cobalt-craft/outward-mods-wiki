# Beastwhispering dev verbs — the command channel

Beastwhispering can be driven at runtime by writing **verbs** to a text file. This is a tester's and
tuner's tool — most of it drives internal state and dumps diagnostics — but a few verbs (`tame`,
`feed`, `recall`, `reloadcfg`) are handy in ordinary play too.

## The command channel

The mod watches **`BepInEx/config/bw_cmd.txt`**. Write a verb line into it and the mod runs that verb
on its next poll — even while the game is **paused** (the poll runs on unscaled time). Results go to
the logs (`BepInEx/LogOutput.log`, and Unity's own `output_log.txt`).

- Write **`help`** (or any unknown verb) to have the mod list every registered verb.
- Some verbs take arguments, space-separated after the verb: `setloyalty 80`, `give 5 Raw Meat`.
- The same verbs are used to retune data tables live and to dump the state of every subsystem.

## Keybinds

Five actions are also bound to keys (rebindable in the `[Keys]` config section):

| Key | Action |
|---|---|
| **F7** | Tame the nearest wild creature |
| **F8** | Feed the pet the first inventory item its diet accepts |
| **F9** | Recall the pet to your feet (also re-forms a bodiless pet) |
| **F10** | Run the compute-layer self-test |
| **F12** | Dump a `[DIAG]` state snapshot to the log |

## Pet lifecycle & control

| Verb | What it does |
|---|---|
| `tame` | Tame the nearest wild creature (the F7 action). |
| `recall` | Recall the pet to your feet; re-forms a bodiless pet. |
| `release` | Release the pet — full teardown, ends the bond, clears the save. |
| `resummon` | Re-form the pet's body now. |
| `reformtest` | Force the re-form path (diagnostic). |
| `petdown` | Down the pet via the real downed flow (tests the re-form-after-defeat path). |
| `petheal` | Heal the pet to full. |
| `pethealth` | Report the pet's current/max HP. |
| `petstatus` | One-screen dump of the pet's whole state (loyalty, hunger, comfort, stance, combat). |
| `pettab` | Open the injected Companion menu tab directly. |
| `petpanel` | Dump the Companion-tab display model to the log (data check with no menu open). |
| `petsound` | Play the species attack/hurt/death sounds in sequence. |

## Feeding & items

| Verb | What it does |
|---|---|
| `feed` | Feed the pet the first accepted inventory item (F8). |
| `feeditem [force] <name-or-id>` | Feed a specific item through the real feed pipeline. |
| `givechow [species]` | Spawn a species's taming chow into the pouch. |
| `givescroll [species]` | Spawn a species's taming-recipe scroll into the pouch. |
| `giveblanket [heat\|cool]` | Spawn a consumable blanket. |
| `givefeather [loyalty\|quality] [qty]` | Spawn Pearlbird feathers of a given quality. |
| `hr <0-5> [seconds]` | Stage a Health Recovery regen on the pet (feed-free). |
| `giveblanket` / `givewater` | Spawn a blanket / a filled waterskin (water via the shared item verbs). |

## Skills & progression

| Verb | What it does |
|---|---|
| `learnskills` | Teach the player all ten Beastwhispering skills at once. |
| `gift` | Cast the Gift skill body (skill-free). |
| `engage` / `disengage` / `petcommand` | The Command Pet skill's two orders and the toggle. |
| `special` | Fire the pet's Hunt as One signature attack. |
| `huntasonesync` | Diagnostic for the Hunt as One / pet-special cooldown sync. |
| `synergy <0-4>` | Set the Synergy stack count for testing. |
| `ftk` | Cast For the Kill (skill-free). |
| `setcourage <0-5>` | Set the species-relic stack count. |
| `travelcheck` / `travelcross` | Inspect / drive the Wild Unknown region-travel gate headlessly. |
| `marenstatus` / `marenspawn` / `marendespawn` / `marenhere` | Inspect, spawn, despawn, or re-place the trainer NPC (`marenhere` stamps her at your current spot). |

## Diagnostics & dumps

Most subsystems have a `…dump` verb that prints its table, its live resolution, and the current state
— the fastest way to see why a data-driven feature is or isn't firing.

| Verb | Subsystem |
|---|---|
| `diag` | Consolidated `[DIAG]` snapshot (scene, vitals, combat, nearby AI, pet). |
| `statdump` | Pet stats: captured vs loyalty-tuned vs the anchor's live readback; damage scale. |
| `dietdump` | Diet table & feed resolution. |
| `tamingdump` | Taming-food table & registration. |
| `tempdump` | Temperature / comfort / blanket state. |
| `mealdump` | Food-hex meal history & next-hit mix. |
| `sigildump` | Sigil-synergy registry & detection. |
| `geardump` | Equipped-gear effects. |
| `weatherdump` | Weather-food buff state. |
| `scentdump` | Scent-sense table & tracker. |
| `scavengedump` | Scavenge-bonus table & armed fill. |
| `giftdump` | Gift drop table & loyalty-lerped mix. |
| `fletchdump` | Feather-fletch registration & held-ammo state. |
| `buffdump` / `bufffooddump` | Bond buffs / buff-food state. |
| `synergydump` / `ftkdump` / `killfavordump` / `tauntdump` | Synergy, For the Kill, kill-favor, taunt state. |
| `boltdump` / `firebolt` | Ranged-special rig census / headless test fire. |
| `bracedump` / `brace` | Brace stance state / force-enter. |
| `echodump` / `echo` | Skill-echo roster / trigger. |
| `warddump` / `lanterndump` | Ward-share / lantern-share state. |
| `foodcats` | The 8 food-category tags → live tag names/UIDs. |
| `speciesaudit` | Cross-table check for tameable-but-missing rows. |
| `statusicons` | Pet status-icon pipeline (inputs → wanted → the player's actual statuses). |
| `caravandump` / `caravanhere` / `caravanspawn` | Caravanner scent-sense diagnostics. |
| `musicdump` / `musiccheck` | Combat-music state snapshot. |
| `recipedump` | Crafting-recipe registration & learned state. |
| `registrydump` | Write item/status/scene registries to `BepInEx/config/bw_registries/` (feeds `bwspecies check`). |
| `donorspot` | Log the current scene's AI roster as pastable donor-scene lines. |
| `anchorstatus` / `anchorforce` / `anchorclear` / `anchorphys` / `anchortest` | Anchor state & weld diagnostics. |
| `visdump` / `visfix` | Puppet draw-readiness dump / repair. |
| `animdump` / `posdump` / `compdump` / `drifttest` | Animator / position / component / drift diagnostics. |
| `combatcheck` | Combat-state readout. |
| `sniff` | Force a scent scan now. |
| `selftest` | Run the compute-layer self-test (F10). |

## Retune tables live (`reload…`)

These re-read a data table's config override and re-apply on the spot, no relaunch (item *registration*
is still boot-time — a brand-new species needs a relaunch):

`reloadcfg` (the whole `.cfg`), `reloadattacks`, `reloadcomfort`, `reloaddiets`, `reloadtaming`,
`reloadfoodhexes`, `reloadsigils`, `reloadgear`, `reloadweather`, `reloadscents`, `reloadgifts`,
`reloadgrowth`, `reloadftk`, `reloadbuffs`, `reloadbufffoods`, `reloadscavenge`, `reloadfletch`,
`reloadskillechoes`, `reloadyaw`.

**`reloadcfg` is the important one:** BepInEx has no config file-watcher, so any raw edit to
`cobalt.beastwhispering.cfg` does nothing until you run `reloadcfg` or relaunch.

## Expedition & templates

| Verb | What it does |
|---|---|
| `expedition <scene\|species>` | Run a donor-region round trip to cache a region-only creature's body. |
| `harvest` | Harvest a donor body for a species. |
| `tamecached` | Tame from a cached body template. |
| `templateprobe` / `templateclear` | Inspect / clear the body-template cache. |

## Test-state & staging (shared ForgeKit verbs)

Beastwhispering inherits the **CommonVerbs** pack from [ForgeKit](../../kits/forgekit.md), so these
answer on `bw_cmd.txt` too. They spawn items, teleport, stage combat, and manipulate character state
for unattended test sessions.

| Verb | What it does |
|---|---|
| `give [pouch\|bag\|ground] [qty] <name-or-id>` | Spawn any item. |
| `drop [qty] <name-or-id>` | Drop an item on the ground ahead of you. |
| `useitem <name-or-id>` | Use the first matching inventory item through the real Use pipeline. |
| `givewater [type]` | Spawn a filled waterskin (clean/river/salt/rancid/…). |
| `givemoney <n>` | Add silver. |
| `equip <name-or-id>` / `unequip <slot>` | Equip / unequip gear (spawns the item if absent). |
| `learnskill <name-or-id>` / `unlearnskill <name-or-id\|all>` | Teach / forget any skill. |
| `resetcooldowns` | Reset every learned skill's cooldown. |
| `teleport <x> <y> <z>` / `goto <scene> [spawn]` | Move the player. |
| `settime <hour>` | Set the game clock. |
| `sethp <pet\|player> <n\|n%>` | Set health (floored at 1 — never a death path). |
| `setloyalty <0-100>` | Set pet loyalty (0 needs `force`). |
| `sethunger <0-1.5>` | Place the hunger clock. |
| `simskip <seconds>` | Fast-forward the pet sim (decay, drains, expiries all apply). |
| `aggro <me\|pet> [name]` / `pacify [radius]` | Force / clear enemy aggro. |
| `killnearest [species] [radius]` | Overkill the nearest wild creature through the real damage pipeline. |
| `combatcheck` / `combatclear` | Read / clear combat state. |
| `scenedump` / `statusdump` / `skydump` / `groundprobe` / `ragdolldump` / `psdump` / `combatmgrdump` / `keybinds` | Engine-state dumps. |

## See also

- [Config reference](./config-reference.md) — every `.cfg` key `reloadcfg` re-reads
- [Data manifests](./data-manifests.md) — the tables the `reload…` verbs retune
- [ForgeKit](../../kits/forgekit.md) — the command channel & shared CommonVerbs
- [Beastwhispering overview](./README.md) · [Mods index](../README.md)
