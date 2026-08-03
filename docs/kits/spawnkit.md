# SpawnKit — spawn any creature, anywhere

**SpawnKit** is a kit for Outward that spawns creatures at runtime as **real vanilla hostiles** — and
an in-game menu that lets you do it yourself. Press **F11**, pick a creature, and it appears beside
you. It's for players who want to stage fights, and for modders who want a one-line creature spawn.

> **Alpha.** The core spawn / despawn / AI loop is solid, but you will hit rough edges (see *Known
> limitations*): roughly a quarter of the creature roster can't be fetched yet, spawning is
> host-driven in co-op, and there's an occasional cosmetic terrain glitch. Back up saves you care
> about before experimenting.

**At a glance**
- Type: gameplay tool + reusable library (kit)
- GUID: `cobalt.spawnkit`
- Requires: BepInEx 5 (Outward's Mono branch), plus [ForgeKit](./forgekit.md),
  [DonorKit](./donorkit.md), [CompanionKit](./companionkit.md), [NetKit](./netkit.md), and
  [AggroKit](./aggrokit.md) (CompanionKit pulls it in)
- Config: `BepInEx/config/cobalt.spawnkit.cfg`
- Commands: `BepInEx/config/SpawnKit_cmd.txt`

## For players

SpawnKit gives you a real vanilla hostile — not a dummy or a stand-in. It wanders, it aggros, it
fights, it drops its normal loot, and it dies like the wild version, because it *is* the wild
version. Spawn a pack of hyenas to train on. Drop a Tuanosaur next to a bandit camp and watch them
fight each other. Restock a region you've already cleared out.

Spawns are **ephemeral**: they never enter your save file, and they vanish when you leave the area.

### Using the menu

Launch the game, load a save, and press **F11** for a draggable window listing every creature you
can spawn.

- **●** means the creature is ready — it spawns instantly.
- **○** means it's cold: the first spawn pauses 1–6 seconds while SpawnKit fetches its body (see
  *How it works*). Click **Prewarm** to pay that cost up front instead.
- Set a count, hit **Spawn**, and use **Despawn All** / **Kill All** to clean up.

The key is rebindable (`[Menu] MenuKey`; `Backslash` is a classic debug-menu choice). The menu closes
on **Esc**, on **F11** again, or when you open a vanilla menu.

### Installing

Installing through a mod manager pulls in every dependency automatically. By hand, drop SpawnKit and
its dependency kits — `ForgeKit/`, `DonorKit/`, `AggroKit/`, `NetKit/`, `CompanionKit/`, `SpawnKit/` — side by side
into `BepInEx/plugins/`. If BepInEx logs a dependency error and refuses to load SpawnKit, one of the
others is missing. See [Installing](../installing.md) for the full setup.

## How it works

Outward has no way to create a creature from nothing — every creature is baked into the scene it
lives in. So SpawnKit borrows one: it quietly loads the scene a species lives in, copies a live
creature out of it, gives the copy a fresh identity, drops it on solid ground near you, and unloads
the scene again. That's the 1–6 second pause on a cold spawn, and it's why the result behaves
perfectly — it's the same creature the game itself would have spawned, not an imitation.

The creature roster comes from **[DonorKit](./donorkit.md)**, which owns the table mapping
each species to the scenes it can be found in — the harvest engine that fetches them, the expedition
tier for region-only species, and the session **template store** SpawnKit's
prewarm cache rides on: SpawnKit's templates live in the store's *spawn kind*, while all the spawn
policy (normalization, the rig reject, the LRU cap) stays SpawnKit's. A third mod can even add a
spawnable species without touching SpawnKit, via `DonorKit.TemplateStore.RegisterSpecies` /
`RegisterTemplate`. `spawnlist` prints the roster. You can add or fix an entry
without rebuilding anything by dropping a `DonorScenes.txt` into `BepInEx/config/`:

```
Hyena=Emercar_Dungeon5,ChersoneseDungeonsSmall
Alpha Tuanosaur=Hallowed_Dungeon3,HallowedMarshNewTerrain
```

**Region-only creatures need an extra step.** Open-world regions and towns are too big to load in the
background — trying it crashes the game, so SpawnKit refuses to. Creatures that live *only* out in a
region (Pearlbird, Torcrab, Coralhorn and friends) therefore can't be fetched the usual way; their
rows are labelled `(expedition-only)` and a cold spawn fails with a clear message. To stock them, run
an expedition: SpawnKit takes the whole party through a loading screen to the donor region and back
(saving on each leg), and from then on that creature spawns like any other. Use
`spawnexpedition <species>`, or the menu's expedition button.

## Settings

`BepInEx/config/cobalt.spawnkit.cfg`, created on first launch:

| Setting | Default | What it does |
|---|---|---|
| `[Spawner] Enabled` | `true` | Master kill-switch. Off = no new spawns (cleanup still works). |
| `[Spawner] AllowSpawnInRoom` | `false` | Override the co-op safety gate: spawn in a room even when a peer *doesn't* have SpawnKit. Those peers see nothing (the spawn is an invisible ghost to them, and locks the host in unwinnable combat), so it's off by default. Solo play never consults this. |
| `[Spawner] MaxActiveSpawns` | `8` | Cap on live spawns at once (pending spawns count). |
| `[Spawner] SpawnDistance` | `5` | How far from you they appear, in meters. |
| `[Spawner] MaxCachedTemplates` | `12` | How many creature bodies stay in memory. Higher = more instant spawns, more RAM. `0` = unlimited. |
| `[Spawner] DefaultCorpsePolicy` | `Vanilla` | `Vanilla` leaves the corpse and its loot. `NoBody` deletes the corpse (**and its loot**) after death. |
| `[Spawner] DefaultCorpseLingerSeconds` | `0` | Seconds a `NoBody` corpse lingers (death-anim time) before it's removed. |
| `[Spawner] ForceLootableEnabled` | `true` | Re-enable a disabled loot component on a spawn so its corpse is lootable. Doesn't invent loot — a creature with no drops still drops nothing. |
| `[Menu] EnableMenu` | `true` | The in-game spawn menu. |
| `[Menu] MenuKey` | `F11` | Key that toggles the menu. |
| `[Expedition] EnableExpeditions` | `true` | Allow fetching a region-only species via a real loading-screen round trip. Off = those rows only spawn if something already cached the body. |
| `[Expedition] ConfirmMenuExpedition` | `true` | Require a second click in the menu to launch an expedition (a stray click shouldn't teleport the party and write a save). |
| `[Coop] EnableCoopSpawns` | `true` | Replicate host spawns to guests running SpawnKit. Off = spawns refuse in a co-op room. |
| `[Coop] DespawnOnMirrorFailure` | `true` | If a guest can't mirror a spawn, despawn it everywhere (safer than leaving a ghost). |
| `[Coop] MirrorQueueTimeoutSeconds` | `20` | How long a guest holds a received spawn (waiting for its scene) before giving up. |
| `[Coop] VerboseNet` | `true` | Extra guest-side `[MIRROR]` mirror-queue logging (duplicate-drop / unknown-uid lines). The per-message co-op send/receive lines are [NetKit](./netkit.md)'s — those are governed by `[Net] VerboseNet` in NetKit's cfg. |
| `[Coop] HeartbeatSeconds` | `30` | **Deprecated and ignored.** The co-op heartbeat now rides NetKit's `[Net] HeartbeatSeconds`; this key is kept only so an existing value doesn't error. A non-default value logs a warning pointing at NetKit's cfg. |

BepInEx has no config file-watcher, and SpawnKit registers no config-reload verb, so an edit to this
file takes effect on the **next launch**. A few settings — the corpse policy and cached-body cap —
apply to *new* spawns only.

### Example configuration

`BepInEx/config/cobalt.spawnkit.cfg` — created on first launch. Excerpt:

```ini
[Spawner]
Enabled = true
MaxActiveSpawns = 8
SpawnDistance = 5

[Menu]
EnableMenu = true
MenuKey = F11

[Expedition]
EnableExpeditions = true
ConfirmMenuExpedition = true

[Coop]
EnableCoopSpawns = true
```

Dev commands run from `BepInEx/config/SpawnKit_cmd.txt`, and the species→scene table is
config-overridable by dropping `BepInEx/config/DonorScenes.txt` (see *How it works*). The
shared-settings overlay is `config/shared/cobalt.spawnkit.cfg.overlay` (see `config/README.md`).

## Commands

Everything the menu does is also driven by writing a line into `BepInEx/config/SpawnKit_cmd.txt` — it
runs on the next poll, even while the game is paused. Useful for scripting, or if a keybind conflicts.
Unknown verb or `help` lists them all.

| Verb | What it does |
|---|---|
| `spawn <species> [count]` | Spawn 1 or more beside you. Names may contain spaces: `spawn Armored Hyena 3`. |
| `spawnex <species> [dist=N] [life=N] [faction=Name] [owner=tag] [body=vanilla\|none] [linger=N] [keepquest]` | One spawn with the full per-spawn option surface. |
| `despawnall [kill]` | Remove every spawn. `kill` gives them a real death (animation + loot) instead of deleting them. |
| `despawnowner <tag> [kill]` | Remove only spawns tagged with `<tag>`. |
| `spawnlist` / `spawndump` | List spawnable creatures / show what's live right now. |
| `spawnprewarm <species>` / `spawnprep <scene>` | Pay the body-fetch cost now so later spawns are instant. |
| `spawnclearcache` | Drop cached bodies (frees memory; next spawn re-fetches). |
| `spawnexpedition <species> [spawn] [force]` | Fetch a region-only species via a loading-screen round trip; `spawn` also spawns it on return. |
| `expeditionreset` | Force a stuck expedition guard open (the fix for "the spawn menu stopped opening"). Doesn't teleport anyone. |
| `spawnmenu` | Toggle the in-game menu regardless of `MenuKey`. |
| `lootprobe` | Per-spawn corpse-loot diagnosis. |
| `bonedump` | Skeleton/rig diagnosis for a spawned body (the "stretched to a vertical line" failure mode). |
| `skcoopdump` | Co-op state on this machine: handshakes, mirrored spawns, message counters. (More co-op dev verbs under `help`: `skinject` / `skfail` / `skdrop` / `skresync` / `skgone` / `skstream` / `skfollow`.) |
| `selftest` | Sanity-check the install; look for `[SELFTEST] … DONE` in the log. |

SpawnKit also registers ForgeKit's shared [CommonVerbs](./forgekit.md) pack on the same channel, so
`give`, `goto`, `teleport`, `settime`, `learnskill`, `scenedump` and the rest are available here too.

## Known limitations

- **Some creatures don't work.** Roughly a quarter of the rows in the list can't be fetched — their
  body isn't in the scene the table expects, or they're region-only and not stocked yet. You get a
  clear message in the log rather than anything harmful.
- **Co-op is host-driven.** With `[Coop] EnableCoopSpawns=true`, the host spawns and every guest
  running SpawnKit mirrors the creature as a real networked enemy — it moves, fights, dies and drops
  loot for everyone. Guests can't start spawns themselves (refused cleanly). If any player in the room
  *doesn't* have SpawnKit, in-room spawns refuse and the log names who, because to them the spawn
  would be an invisible ghost. `[Spawner] AllowSpawnInRoom=true` overrides that, knowingly; the whole
  co-op wire rides [NetKit](./netkit.md).
- **⚠ Everyone in a co-op room needs the SAME SpawnKit build (0.4.0 or newer on all boxes).** The
  0.4.0 co-op wire changed shape when the spawn lifecycle moved onto NetKit's replicated-record
  store, and it is deliberately not backward compatible. Mixing 0.4.0 with an older build doesn't
  half-work — the handshake reports the peer as incompatible and co-op spawning stays off toward
  them, which is the intended outcome. Update everyone together and it's a non-event.
- **A rare visual glitch.** Very occasionally, after a lot of spawning in one session, a patch of
  ground can go invisible (you can still walk on it) — changing zone and coming back repairs it.

### Troubleshooting

| Symptom | What it means | What to do |
|---|---|---|
| `No spawnable species matches '<x>'` | Not in the table | Check `spawnlist`; add a line to `BepInEx/config/DonorScenes.txt` |
| `has NO additive donor — every candidate is an oversized region/town scene` | A region-only creature, not stocked yet | Run `spawnexpedition <species>` (or the menu's expedition button) |
| First spawn of a creature pauses, later ones don't | Normal — that's the body fetch | Use `spawnprewarm` to pay it early |
| No mods load at all, no crash log | Wrong Steam branch (IL2CPP instead of Mono) | See [Installing](../installing.md) |

## For modders

SpawnKit is also a library. Reference it and declare the dependency:

```csharp
[BepInDependency(SpawnKit.Plugin.GUID)]
```

Then spawning is one line:

```csharp
SpawnHandle h = Spawner.Spawn("Hyena");
```

Everything else is optional — per-spawn customization rides a `SpawnOptions`, and the returned
`SpawnHandle` is your window onto the creature's lifecycle:

```csharp
Spawner.Spawn("Alpha Tuanosaur",
    new SpawnOptions
    {
        Faction = Character.Factions.Tuanosaurs, // null = the donor's own faction
        Distance = 12f,                          // ring-probe radius (clamped 1–50m)
        // Position = somewhereExact,            // skip the ring probe entirely
        LifetimeSeconds = 120f,                  // auto-despawn clock (null = forever)
        Corpse = Core.CorpsePolicy.NoBody,       // destroy the corpse after death (null = the
                                                 //   [Spawner] DefaultCorpsePolicy config;
                                                 //   NoBody takes the LOOT with it — see traps)
        CorpseLingerSeconds = 8f,                // death-anim time before the NoBody destroy
        OwnerTag = "mymod",                      // namespace your spawns (see DespawnAll)
        // StripQuestEvents = false,             // default true — leave it unless you KNOW
        ConsumerData = "objective:3",            // opaque string; reaches co-op guests (below)
        OnBeforeActivate = (go, ch) =>           // last touch before the creature wakes up
        {
            go.AddComponent<MyObjectiveMarker>();
        },
    },
    onReady: handle =>
    {
        if (!handle.IsAlive) { /* handle.FailReason says why */ return; }
        handle.OnDied += _ => DropReward();
        handle.OnDespawned += _ => CleanupMarker();
    });
```

The full surface (`SpawnKit.Spawner`, all static):

| Call | What it does |
|---|---|
| `Spawn(species, options?, onReady?)` | Spawn ONE creature; returns a `SpawnHandle` immediately (`Pending` → `Alive`/`Failed`; sync refusals return already-`Failed` with a `FailReason`). One call = one creature — loop for more; a cold-species loop still costs one fetch (in-flight acquires dedup). |
| `Despawn(handle, kill=false)` | Remove one spawn: `kill` = real damage pipeline (death anim + loot, → `Died`); silent = destroy (→ `Despawned`). Cancels a still-`Pending` handle. |
| `DespawnAll(ownerTag=null, kill=false)` | Sweep by owner tag. `null` = EVERYTHING — that's the operator verb; mods should pass their own tag. |
| `Prewarm(species, onDone?)` / `IsPrewarmed(species)` | Pay/check the body fetch off your hot path. |
| `CanMintNow(species)` | True if a spawn can mint a body right now, from either cache (prewarm or expedition) — the question UI actually wants. |
| `IsExpeditionOnly(species)` / `IsExpeditionCached(species)` | Whether a species can only be stocked by an expedition, and whether one already has. |
| `Expedition(species, onDone?, force=false)` | Go fetch an expedition-only species (party round trip, saves on each leg). Host/offline only. |
| `Active(ownerTag=null)` | Snapshot of tracked handles (Pending + Alive). **Host-side** — on a co-op guest this is empty (see below). |
| `Replicas()` | Snapshot of the mirrored creatures on THIS machine — the guest-side counterpart of `Active()`. Empty on the host/solo. |
| `OnMirrored` | Event: fires on a guest each time a mirrored creature finishes arriving. |
| `SpeciesKeys()` / `ResolvedDonorName(species)` | The spawnable roster, and the real donor creature a species key resolves to (once revealed). |

### Two extension seams

**`OnBeforeActivate(GameObject, Character)`** runs on the freshly built creature *before it wakes
up* — after SpawnKit has finished stamping its identity, and before the game's own initialization
runs. That ordering is the whole point: anything the creature's own start-up reads (a marker
component you want present from frame one, a loot-table edit, an AI tweak) has to be in place
*before* activation, because by the time `onReady` fires it is already a frame too late. Keep it
cheap, don't activate the object yourself, and note that a throwing hook is logged and skipped —
it never aborts the spawn. Host-side only.

**`ConsumerData`** is an opaque string you attach to a spawn. In co-op, SpawnKit already mirrors
your creature onto every guest for free — but until now the guest had no way to tell *which* of
your creatures it was looking at, because delegates and object references cannot cross the
network. `ConsumerData` can: whatever you put on it comes back on the guest, unchanged.

```csharp
// On a guest — find your own mirrored creatures.
Spawner.OnMirrored += info =>
{
    if (info.ConsumerData == "objective:3") MarkObjectiveVisible(info.Uid);
};

foreach (ReplicaInfo r in Spawner.Replicas())
    Log.LogMessage($"{r.SpeciesKey} {r.Uid} state={r.State} data='{r.ConsumerData}'");
```

`ReplicaInfo` is plain read-only data (uid, species, state, view id, your string) — deliberately
*not* the `Character`. Guests observe mirrored creatures; the host drives them. Keep the string
short: it travels with the creature every time the spawn is synchronized. A peer running an older
SpawnKit simply ignores it, and a spawn from an older peer reads back as `""`.

### Trap table — read before shipping on this

| Trap | Rule |
|---|---|
| `handle.Character` can go **Unity-null** under you (scene unload, despawn) | Check `handle.IsAlive` before every use; never cache the `Character` across scene loads. |
| Spawns are **ephemeral by design** | Scene unload = `OnDespawned`, and saves skip `SK_` UIDs both directions. Don't build persistence on top. |
| A **cold species blocks ~1–6s** before `onReady` | `Prewarm` on your init path if the spawn must feel instant. |
| `MaxActiveSpawns` is **global across all consumers** (pending spawns count too) | Tag your spawns (`OwnerTag`) and never call `DespawnAll(null)` from a mod — you'd sweep other mods' spawns. |
| A consumer's **own cap can bind long before SpawnKit's**, and users will blame SpawnKit | If you enforce a private cap, say so in that key's description and log the block naming BOTH keys and BOTH files — otherwise "it stops at 6 whatever I set `MaxActiveSpawns` to" is the bug report you get (it is the one we got: DangerousRoads' `[Wave] MaxOwnActive`, 2026-08-02). |
| Callbacks run on the **main thread inside the watch tick** | Keep them cheap. A throwing callback is isolated (swallowed + logged) — it won't break SpawnKit, but you won't get a second chance either. |
| **Co-op clients always get `Failed/NotMaster`** — guests never initiate | Design your feature so only the host drives spawning. Host-side in-room spawns work when every peer handshakes; otherwise the refusal names the unmodded peer. |
| An **unmodded guest in the room** blocks in-room spawns (`PeersNotReady`) | That's the feature: a spawn would be an invisible ghost to them. `AllowSpawnInRoom=true` overrides, knowingly. |
| A guest's **first mirror of a species runs its own ~1s donor harvest** | Prewarm on both machines if it matters. |
| Late-join / scene-return flush covers **Alive spawns only** | A guest joining never sees pre-existing corpses; a `NoBody` corpse removal can destroy a loot container a guest has open (same as a scene unload mid-loot — vanilla tolerates it). |
| `StripQuestEvents = false` on a quest-flagged creature **fires real quest events on death** | The default (true) exists because live donors really do carry quest triggers — leaving it on is what keeps spawns from mutating quest state. |
| `CorpsePolicy.NoBody` **destroys the loot bag with the body** | The corpse GameObject IS the loot container — never combine NoBody with a drop-loot expectation. |
| Ring geometry (step angles, elevation band, navmesh snap) is **not configurable** | By design. If you need exact placement, pass `SpawnOptions.Position` and own the spot yourself. |
| The species roster and creature supply are **DonorKit's**, not SpawnKit's | An expedition-only species fails cold spawns until an expedition stocks it. |
| `Active()` is **empty on a co-op guest** | It is the host's registry. A guest sees mirrored creatures through `Replicas()` / `OnMirrored`, which carry `ReplicaInfo`, not handles. |
| `OnBeforeActivate` runs **before the creature is initialized** | Half the `Character` is not set up yet. Add components and edit data there; do anything that needs a *live* creature in `onReady` instead. |
| `ConsumerData` is **not a channel** | It is a fixed label attached to one spawn, sent when that spawn is synchronized. It never updates in place — if the value must change over time, replicate that yourself. |

## See also

- [Kits index](./README.md)
- [CompanionKit](./companionkit.md) — the companion spine
- [DonorKit](./donorkit.md) — the creature roster, harvest engine and template store; its expedition trips also stock bodies SpawnKit can adopt
- [NetKit](./netkit.md) — the co-op layer host spawns replicate over
- [ForgeKit](./forgekit.md) — the command channel and shared dev verbs
- [Installing](../installing.md)
- [Wiki home](../README.md)
