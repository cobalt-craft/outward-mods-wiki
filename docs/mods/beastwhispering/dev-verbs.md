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
| `tame` (alias `clonevisual`) | Tame the nearest wild creature (the F7 action). |
| `recall` | Recall the pet to your feet; re-forms a bodiless pet. |
| `release` | Release the pet — full teardown, ends the bond, **deletes the pet's save file**. Irreversible, so it is guarded: the first call only warns and arms a 30-second window; `release confirm` (or calling `release` again inside the window, at least a second later — two `release` lines in one command-file write do NOT confirm each other) is what actually does it. Tune the guard with `[Systems] ReleaseConfirmLoyaltyThreshold`. |
| `resummon` | Re-form the pet's body now. |
| `reformtest` | Force the re-form path (diagnostic). |
| `petdown` | Down the pet via the real downed flow (tests the re-form-after-defeat path). |
| `petheal` | Heal the pet to full. |
| `pethealth` | Report the pet's current/max HP. |
| `bandage` | Apply the first bandage in your inventory to the pet (the right-click "Bandage" action's pipeline, headless). |
| `petname <name>` | Name the active pet (`petname` with no argument, or `petname --clear`, removes the name so it shows its species again). Same path the Companion tab's rename field uses. |
| `petnameui [open\|close\|dump]` | Drive the Companion tab's rename field without a mouse, and log its UI state (selection, focus, raycaster). `dump` is the default. Diagnostic — use it when the header does not open an edit field. |
| `petstatus` | One-screen dump of the pet's whole state (loyalty, hunger, comfort, stance, combat). |
| `pettab` | Open the injected Companion menu tab directly. |
| `petpanel` | Dump the Companion-tab display model to the log (data check with no menu open). |
| `petsound` | Play the species attack/hurt/death sounds in sequence. |
| `vocals` | Toggle the body-synced species attack vocal live. |
| `bodycensus` | List every live companion body on this machine (id, species, origin, age, claim state). |

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

Remedies go through this same pipeline: feeding the pet an Antidote, a bandage, a Panacea or a drink
of water applies that item's *own* vanilla cleanse to the pet, exactly as using it on yourself would.
A remedy is not food, so no species diet has to list it — but an item that would cure nothing is
never consumed. `curedump` (below) shows what each carried item would cure right now.

## Skills & progression

| Verb | What it does |
|---|---|
| `learnskills` | Teach the player all ten Beastwhispering skills at once. |
| `gift` | Cast the Gift skill body (skill-free). |
| `engage` / `disengage` / `petcommand` | The Command Pet skill's two orders and the toggle. |
| `special` | Fire the pet's Hunt as One signature attack. |
| `petsigil <Fire\|Frost\|Air\|Blood> [here]` | The pet lays its own sigil circle at its feet (`here` drops it at yours). |
| `huntasonesync` | Diagnostic for the Hunt as One / pet-special cooldown sync. |
| `synergy <0-4>` | Set the Synergy stack count for testing. |
| `ftk` | Cast For the Kill (skill-free). |
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
| `sigildump` / `sigilspawn` | Sigil-synergy registry & detection / place a sigil world item at your feet. |
| `geardump` | Equipped-gear effects. |
| `weatherdump` | Weather-food buff state. |
| `scentdump` | Scent-sense table & tracker. |
| `scavengedump` / `scavengesim` | Scavenge-bonus table & armed fill / simulate the nearest container's drop tables many times, with and without the pet's scavenge dice (opens nothing, changes nothing). |
| `giftdump` | Gift drop table & loyalty-lerped mix. |
| `fletchdump` | Feather-fletch registration & held-ammo state. |
| `buffdump` / `bufffooddump` | Bond buffs / buff-food state. |
| `synergydump` / `ftkdump` / `killfavordump` | Synergy, For the Kill, kill-favor state. |
| `boltdump` / `firebolt` | Ranged-special rig census / headless test fire. |
| `bracedump` / `brace` / `tauntdump` / `taunt [name]` | Brace stance state (including its aggro-pin) / force-enter / taunt pipeline state / force a short taunt onto the nearest (or named) enemy. |
| `hitfxdump` | What the equipped weapon would proc on a Hunt as One / For the Kill strike. |
| `echodump` / `echo` | Skill-echo roster / trigger. |
| `warddump` / `lanterndump` | Ward-share / lantern-share state. |
| `bandagedump` | Bandage-to-pet pipeline (resolved item/status, live anchor state, last action). |
| `curedump` | Status cures: the switch, whether the live water dispenser is up (which of the two drink paths is live), the anchor's live statuses, and — per inventory item — what it can cleanse and whether it would cure anything *right now*. An item that would cure nothing is never consumed. |
| `foodcats` | The 8 food-category tags → live tag names/UIDs. |
| `speciesaudit` | Cross-table check for tameable-but-missing rows. |
| `statusicons` | Pet status-icon pipeline (inputs → wanted → the player's actual statuses). |
| `dotauradump` | Pet damage-over-time auras: the switch and tick count, the anchor's live statuses and the auras they imply, and per row whether that status's FX actually resolves (a status prefab carrying no FX prefab is the one silent failure). Force one by hand with CompanionKit's `aura <key> on\|off\|clear`. |
| `caravandump` / `caravanhere` / `caravanspawn` / `caravanreroll` | Caravanner scent-sense diagnostics / re-run the caravanner's own vanilla spawn roll (host or solo only). |
| `musicdump` / `musiccheck` | Combat-music state snapshot. |
| `recipedump` | Crafting-recipe registration & learned state. |
| `registrydump` | Write item/status/scene registries to `BepInEx/config/bw_registries/` (feeds `bwspecies check`). |
| `donorspot` | Log the current scene's AI roster as pastable donor-scene lines. |
| `anchorstatus` / `anchorforce` / `anchorclear` / `anchorphys` / `anchortest` | Anchor state & weld diagnostics. |
| `visdump` / `visfix` | Puppet draw-readiness dump / repair. |
| `yaw <deg>` | Live rig-facing tune for a body that walks sideways/backward (persist the winner in `[Pet] SpeciesYawOffsets`). |
| `animdump` / `posdump` / `compdump` / `drifttest` | Animator / position / component / drift diagnostics. |
| `combatcheck` | Player combat state + the anchor weld health. |
| `groundprobe` | Measure the navmesh under the player, puppet and anchor (Beastwhispering's pet-aware version of the shared verb). |
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
| `harvest <scene> <creature>` | Additively load a donor scene and clone a creature out of it. |
| `tamecached` | Tame from a cached body template. |
| `templateprobe` / `templateclear` | Inspect / clear the body-template cache. |

## Test-state controls

Beastwhispering's own staging verbs — they place the pet's simulation exactly where a test needs it.

| Verb | What it does |
|---|---|
| `setloyalty <0-100>` | Set pet loyalty (dropping to 0 is abandonment and needs `setloyalty 0 force`). |
| `sethunger <0-1.5>` | Place the hunger clock, as a fraction of the hunger day (`1.0` = starving). |
| `setcourage <0-5>` | Set the species-relic stack count. |
| `simskip <seconds>` | Fast-forward the pet sim — decay, drains and expiries all apply honestly. |
| `sethp <pet\|player> <n\|n%>` | Set health, floored at 1 (Beastwhispering's pet-aware version of the shared verb — death still needs real damage). |
| `aggro <me\|pet> [name]` / `pacify [radius]` | Force / clear enemy aggro. |

## Shared verb packs

Three libraries register their own verb packs onto `bw_cmd.txt`, so their verbs answer here without a
second command channel.

**[ForgeKit](../../kits/forgekit.md) CommonVerbs** — generic staging and engine diagnostics:

| Group | Verbs |
|---|---|
| Items | `give [pouch\|bag\|ground] [qty] <name-or-id>` · `drop` · `useitem` · `givewater [type]` · `equip` · `unequip <slot>` |
| World | `teleport <x> <y> <z>` · `goto <scene> [spawn]` (host only) · `settime <hour>` (host only) · `givemoney <n>` · `face` · `moveto` |
| Combat | `combatclear` · `killnearest [species] [radius]` · `swing` · `lockon` · `lockoff` |
| Skills | `learnskill` · `unlearnskill <name-or-id\|all>` · `resetcooldowns` · `castspell` |
| Status | `grantstatus <name> [target=player\|pet] [force]` · `removestatus` |
| Engine dumps | `statusdump` · `pos` · `scenedump` · `skydump` · `combatmgrdump` · `keybinds` · `ragdolldump` · `psdump` |
| Containers | `containerdump [radius]` · `containerroll [filter] [radius] [fresh\|reopen]` (host only) |
| Recovery | `unstick` (alias `unwedge`) — diagnose and recover a wedged session |

The pack's own `sethp` and `groundprobe` are **not** registered here: Beastwhispering ships pet-aware
supersets of both, listed above.

**[SkillKit](../../kits/skillkit.md)** (it has no channel of its own): `castdump` · `castclear` ·
`skillverify` · `skilldump <name>` · `skillitemdump [name]` · `skillkitreloadcfg`.

**[DonorKit](../../kits/donorkit.md)** donor-harvest diagnostics: `photondump` · `audiodump` ·
`audioprune` · `terraindump` · `terrainfix`.

ForgeKit's command channel itself also provides `script` · `scriptcancel` · `scriptstatus` for
running a queued sequence of verbs.

## See also

- [Config reference](./config-reference.md) — every `.cfg` key `reloadcfg` re-reads
- [Data manifests](./data-manifests.md) — the tables the `reload…` verbs retune
- [ForgeKit](../../kits/forgekit.md) — the command channel & shared CommonVerbs
- [Beastwhispering overview](./README.md) · [Mods index](../README.md)
