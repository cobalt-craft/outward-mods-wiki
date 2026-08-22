# SpawnKit — spawn any creature, anywhere

**SpawnKit** is a kit for Outward that spawns creatures at runtime as **real vanilla hostiles** — and
an in-game menu that lets you do it yourself. Press **\\** (backslash), pick a creature, and it appears beside
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

Launch the game, load a save, and press **\\** (backslash) for a draggable window listing every creature you
can spawn.

- **●** means the creature is ready — it spawns instantly.
- **○** means it's cold: the first spawn pauses 1–6 seconds while SpawnKit fetches its body (see
  *How it works*). Click **Prewarm** to pay that cost up front instead.
- Set a count, hit **Spawn**, and use **Despawn All** / **Kill All** to clean up.

The key is rebindable (`[Menu] MenuKey`, default `Backslash` — a classic debug-menu choice). The menu
closes on **Esc**, on **\\** again, or when you open a vanilla menu.

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

## Room-wide warm mirror (co-op)

**The problem.** A cold spawn is cheap on the machine that chose it and brutal on the one that
didn't. When the host spawns a species a *guest* holds no body template for, that guest has to
additively load a whole donor scene on its main thread to mirror the creature — measured in the
field at **3.6 s (Obsidian Elemental), 12.1 s (Ghost) and 14.7 s (Elder Medyse)**, mid-fight, in a
dungeon. Photon's 10 s disconnect timeout sits inside that range, which is why the reported symptom
was a guest "constantly disconnecting during fights". The guest-side backstop (see
`docs/guest-mirror-harvest-testplan.md`) can *refuse* such a spawn, but only the host can avoid
**choosing** it — and for that the host has to know what every guest already holds.

**The mechanism.** Every peer publishes its own warm set. SpawnKit registers one
[NetKit](./netkit.md) replicated store, `warm`, on the `sk` channel with **PeerOwned** authority:
each peer owns exactly one row, the receiver keys that row by the unforgeable sender actor, and
every machine — host included — mirrors every other peer's row. The payload carries the peer's
mintable species, its proven **dead ends**, and its **remaining donor-harvest budget**, so the host
never wants forever against a guest that structurally cannot comply.

- Publishes are **change-latched on the encoded bytes** (`sk.warmset`): a re-publish of an unchanged
  set sends nothing, so the mirror can be as trigger-happy about marking itself dirty as it likes.
- A **30 s refresh** heartbeats each row so a dropped message cannot leave a mirror stale forever,
  and the store **flushes to a late joiner** on its scene-ready edge — nobody has to ask.
- Events (a template cached or evicted, a finished warm, scene-ready, room change) are *latency*; a
  **2 s verification poll** is *correctness*, because the expedition cache has no change seam at all.
- There is deliberately **no `sk.warmed` confirm message**. The row *is* the confirm: it lands
  exactly when the set actually grew. Two paths to the same fact with different arrival orders is
  how you get "the host thinks it's warm and the mirror says cold".
- `sk.want` (master → one actor) is the only other new verb: *"warm this species when it is safe,
  at the head of your queue."* The guest still decides when that is — out of combat, no save pending.

**The host gate.** `[Coop] RoomWarmMode` decides how strictly a spawn is gated on the party rather
than on this box:

| Mode | What the host does |
|---|---|
| `Local` | ignores peers entirely — the pre-co-op behaviour |
| `RoomDegraded` *(default)* | spawns anyway, **names the cold actors in the log**, and asks them to warm the species so the next one is instant |
| `RoomStrict` | refuses the spawn (`FailReason.SpeciesColdOnPeer`, actors named) |

`RoomDegraded` is the default because a road ambush that silently stops arming is a worse experience
than one guest paying a donor load. `RoomStrict` is the setting for a party that would rather see
fewer enemies than have anyone stall mid-fight.

Two verdicts are kept apart on purpose. **`ColdLocally`** means *this* machine can't mint it — not a
peer's fault, and never reported as one. **`SpeciesColdOnPeer`** means the host could have minted it
and a named guest could not; that is the only verdict the room gate owns, and the only one a
`sk.want` can fix. Dev verbs escape per call with a trailing **`force`** token (`spawn`, `spawnex`),
which sets `SpawnOptions.IgnoreRoomWarm` — deliberately not a config key, because a forgotten global
would silently disable the gate for every consumer.

**The fluke path.** A miss is not fatal, it is a request. The host keeps a **want book** per
(actor, species): send, then wait out `[Coop] RoomWarmRequestTimeoutSeconds` (30 s, doubling to
`RoomWarmRequestMaxTimeoutSeconds`) before re-asking, up to `RoomWarmRequestAttempts` times, then
give up once with a single `[MIRROR]` line. A guest that reports the species a **dead end**, or that
has **zero harvest budget left**, is given up on *immediately* — that is a structural no, not a slow
yes. `RoomWarmRequestCap` bounds how many species one `RequestRoomWarm` call may ask of one peer
(3), and the caller's priority order goes to **every** peer, so the room-wide intersection converges
instead of two guests warming disjoint halves.

**Forgiven cold-unsafe.** A guest can still refuse a spawn with `sk.fail(cold-unsafe)` *after* the
room gate was satisfied — an eviction that lost a race, not a lie. The first such refusal for a uid
is **forgiven**: the host logs `[SKNET] cold-unsafe DESPITE room gate`, sends `sk.want`, and keeps
the creature so the guest's own detached acquire can finish. When the guest's set then grows, the
host **re-flushes** the forgiven spawns to it (`[SKNET] re-flushed N forgiven spawn(s)…`); a pure
ledger holds a ~60 s safety net, so a forgiven creature that is never mirrored despawns rather than
lingering as a ghost, and a second refusal for the same uid despawns immediately. See
`SpawnNet.OnFail`. That log line is the Phase B/C regression alarm — in a healthy `RoomStrict`
session it should never appear.

**Invariants worth knowing.**
- **An unmodded or still-loading peer never vetoes.** Only a peer that has actually published a warm
  set participates; anyone else is invisible to the gate, exactly as before.
- **Solo play is byte-identical in every mode.** Zero participating peers ≡ local truth.
- **"The guest is warm for 4 of 13 species" is normal, not a bug.** A guest warms what it has been
  asked for and what it has mirrored; the intersection grows over a session.
- **There is a quiet window right after a join.** Until a fresh guest's row lands and the first
  `sk.want`s are answered, the room-wide intersection is small and `RoomStrict` will arm fewer
  ambushes. That is the gate working.

**Seeing it.** `skwarmdump` prints this machine's set, dead ends and remaining budget, then every
peer's row — `Unmodded` (no compatible SpawnKit), `HelloedNoRow` (helloed, set not in yet), or
`Participating` with its age, species and budget — then the room-wide intersection and the whole want
book. `spawnlist`, the spawn menu and DangerousRoads' `roadsroster` share one glyph key:
**● room-wide warm · ◐ warm here, cold on a peer · ○ cold here**.

**Wire version.** These verbs ship under `sk` channel version **0.5.0**, matched by exact ordinal
comparison. A mixed-build room does not half-work: the handshake reports the peer as incompatible,
says so loudly in the log, and co-op spawning stays off toward it. Update everyone together.

### Warm-mirror settings

`BepInEx/config/cobalt.spawnkit.cfg`:

| Setting | Default | What it does |
|---|---|---|
| `[Coop] RoomWarmMode` | `RoomDegraded` | `Local` / `RoomDegraded` / `RoomStrict` — how strictly a spawn is gated on the whole party being able to mint the species (see above). |
| `[Coop] WarmSetPublishSeconds` | `1` | Minimum seconds between two publishes of this machine's warm set; every dirty event coalesces into one. Raising it costs freshness, not correctness. |
| `[Coop] RoomWarmRequestCap` | `3` | Max species asked of ONE peer per `RequestRoomWarm` call. `0` or negative = uncapped. |
| `[Coop] RoomWarmRequestTimeoutSeconds` | `30` | How long the host waits for a guest to answer an `sk.want` before re-asking. Long on purpose — answering means running a donor load, deferred to a safe moment. |
| `[Coop] RoomWarmRequestMaxTimeoutSeconds` | `120` | Ceiling on the doubling backoff between retries. Values below the timeout are clamped up, never rejected. |
| `[Coop] RoomWarmRequestAttempts` | `3` | How many times the same `sk.want` is sent before the host stops asking for that (peer, species) this session. Minimum 1. |

```ini
[Coop]
RoomWarmMode = RoomDegraded
WarmSetPublishSeconds = 1
RoomWarmRequestCap = 3
RoomWarmRequestTimeoutSeconds = 30
RoomWarmRequestMaxTimeoutSeconds = 120
RoomWarmRequestAttempts = 3
```

> **Built 2026-08-20, not live-verified.** Rows GM10–GM19 in
> `docs/guest-mirror-harvest-testplan.md` are the live gates.

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
| `[Menu] MenuKey` | `Backslash` | Key that toggles the menu (rebindable). |
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
MenuKey = Backslash

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
| `spawn <species> [count] [force]` | Spawn 1 or more beside you. Names may contain spaces: `spawn Armored Hyena 3`. `force` ignores the room-wide warmth gate for this call. |
| `spawnex <species> [dist=N] [life=N] [faction=Name] [owner=tag] [body=vanilla\|none] [linger=N] [keepquest] [force]` | One spawn with the full per-spawn option surface. |
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
| `skwarmdump` | Warm-mirror census: this machine's warm set, every peer's row, the room-wide intersection and the want book (see *Room-wide warm mirror*). |
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
- **⚠ Everyone in a co-op room needs the SAME SpawnKit build (0.5.0 on all boxes).** 0.5.0 added the
  warm-mirror verbs (`sk.warmset` / `sk.warmclr` / `sk.want`) and the `sk` channel version is compared
  by exact ordinal match, so a 0.4.0 peer is reported incompatible and co-op spawning stays off toward
  it — loudly, not half-working. The
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
