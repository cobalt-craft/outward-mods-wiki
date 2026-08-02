# DonorKit — creature bodies from anywhere in the world

Outward's creatures are not spawnable. There is no prefab to instantiate, no registry to ask: every
beast in the game is *baked into the area scene it lives in*. If you want a Pearlbird and you are
standing in a dungeon, the vanilla answer is "walk to the coast".

DonorKit is the answer that does not involve walking. It loads any area of the game *additively, in
the background*, about a second, no loading screen — the creatures in it wake up as live instances —
hands one of them to your code, and unloads the area again. Your mod gets a real, fully-initialised
creature to clone, wherever the player happens to be standing. For the region and town scenes that
are too large to load that way, it owns the **expedition** tier instead: a real, hands-off round
trip through the vanilla loading screen that batch-caches every species the region donates. Either
way the result lands in its **template store** — dormant bodies, cached once per session, served in
milliseconds afterwards.

This is a library for mod authors. Players never interact with it directly; it is installed because
something else needs it.

**At a glance**
- Type: reusable library (kit)
- Requires: BepInEx 5 (Mono branch), [ForgeKit](./forgekit.md), [NetKit](./netkit.md)
- Config: `BepInEx/config/cobalt.donorkit.cfg`
- Commands: `BepInEx/config/DonorKit_cmd.txt`
- Data files: `BepInEx/config/ck_expeditions.txt` (the expedition manifest — the historical name is
  kept so existing installs keep their record)
- No SideLoader dependency

## For players

You do not need to install this on purpose. [CompanionKit](./companionkit.md) (pet bodies) and
[SpawnKit](./spawnkit.md) (creature spawning) both depend on it, so it arrives with them.

What you may *notice* is the second or so of quiet work when a mod needs a creature it has not seen
before — a pet re-forming after a load, or the first spawn of a species. That is a donor load. It is
paid once per species per session; afterwards the body is cached and re-forms are instant.

If something has gone visibly wrong after one of those — audio cut out, a rectangular hole in the
ground, a black screen — that is this kit's problem domain, and the commands below are how you
report it. Every one of those failure modes is a real bug this kit was written to survive, and each
carries its own guard.

## How it works

The recipe, in order, is:

1. **Additively load** the donor area by name (`SceneManager.LoadSceneAsync(name, Additive)`). The
   game itself never does this; nothing stops a mod from doing it.
2. **Freeze it the instant it lands** — every root object is deactivated in the `sceneLoaded`
   callback, which fires after the donor's `Awake` but *before* any of its `Start`/`Update`. No donor
   script ever gets a frame. This is what stops a donor scene from stamping its own skybox, camera or
   weather over the real one.
3. **Run your payload** while the donor is loaded and frozen. You get the live `Character`; you clone
   what you need. The payload is synchronous and must not keep references into the donor — everything
   you do not clone out dies at step 4.
4. **Unload**, then repair: re-tetrahedralize the light probes, restore the skybox, prune the audio
   registries, check the terrain.

Around that sit the guards, and they are most of the code. Each one exists because the un-guarded
version broke a real session:

| Guard | What it stops |
|---|---|
| `DonorPhotonGuard` | Donor `PhotonView`s registering and colliding with the live scene's NPCs (the NPC loses) |
| `DonorSaveGuard` | A save landing mid-harvest and serialising a world state the game can never reach on its own. Deferred, not dropped — the game's own `SetSaveRequired` re-arm replays it moments later |
| `DonorSingletonRestore` | Four self-registering singletons (audio, combat, environment, AI squads) that a donor scene hijacks on `Awake` and destroys on unload |
| `AmbienceGuard` + `DonorAudioRegistry` | Donor ambience left dangling in the persistent audio manager, retried every frame forever (~190 exceptions/sec, sustained) |
| `SoulSpotGuard` | A duplicate soul-spot UID throwing *mid-`Awake`*, leaving a half-initialised creature and a native crash behind it |
| `DonorDisplayRepair` | The black-screen and display-chain clobbers |
| `TerrainGuard` | Terrain tiles left with a destroyed LOD mesh — an invisible rectangle you can still walk on |
| `DonorNetworkInitGuard` | A guest converting donor creatures into network replicas before they can be cloned |

`SkeletonRig` and `RagdollRig` handle the other half: a clone's skinned-mesh bones and ragdoll joints
can reference transforms that die with the donor scene. Unrepaired, the creature renders as a
stretched vertical line, or collapses into a stretched corpse on death.

### The one real budget

Every additive load/unload cycle degrades Unity's global `LightProbesManager`, and after roughly
11–17 cycles the next probe-carrying scene activation crashes inside the engine. `LightProbes
.Tetrahedralize()` after each unload is the mitigation, and it is why cycles are treated as a scarce
resource: batch consumers harvest many species per load rather than one at a time. The running count
is in every `[HARVEST]` completion line.

### The expedition tier

Some creatures live only in open-world regions and towns. Those scenes are too large to load
additively — it crashes — so they are marked oversized and the additive path refuses them. Reaching
them is the expedition tier: a real round trip through
the vanilla loading screen. `ExpeditionHarvest` drives two ordinary area switches (out and back),
runs a capture payload in the donor region while the screen is still black and gameplay paused,
auto-passes both "press any key" gates, restores every player to their marked position, and records
the swept scene in `ck_expeditions.txt`. `ExpeditionOrchestrator` owns policy on top: the
`expedition` verb, the batch capture payload (every species the region donates, one trip),
opportunistic in-place capture when the player naturally walks into such a region, and the
once-per-launch boot auto-warm. `StorySense` holds the story-safety line — never depart the
prologue, never target a scene this character has not visited (a load behind a black screen would
consume its one-shot first-entry beats).

Expeditions are host/offline-only and refuse to start with guests connected unless explicitly
allowed (see `[Expedition] AllowCoop` below — the failure mode is the whole party stranded in the
donor region).

### The template store

Both supply paths land bodies in the kit's session **template store** — one store, two *kinds*,
deliberately different policies:

- **companion-body kind** (`BodyTemplateStore`): dormant puppet sources with captured identity +
  stats. Eager-prunes destroyed bodies, *refuses substitutes on identity lookups* (a cached
  "Coralhorn" body may really be an "Alpha Coralhorn" — fine as a pet's fallback body, a lie as an
  identity), keeps unhealthy rigs (a mis-skinned pet body beats none), and warns as the census
  grows.
- **spawn kind** (consumed by [SpawnKit](./spawnkit.md)): bare dormant creatures normalized for
  minting real hostiles. Opt-in LRU cap eviction, rejects templates whose rig cannot be repaired
  (the harvest chain advances instead).

The kinds share only the fabric (an inactive `DontDestroyOnLoad` holder, a keyed registry, recency
tracking, an orphan-proof clear); every lookup/eviction/refusal policy stays with its consumer,
because unifying them would change behavior — the two caches answer different questions on purpose.

**Third-party seams** — a mod can add spawnable/harvestable species without touching SpawnKit:

```csharp
// Add a donor-table row at runtime: the species becomes visible to every existing pipeline
// (SpawnKit menu/spawn/prewarm, companion harvest chains, the expedition tier's classification).
DonorKit.TemplateStore.RegisterSpecies("My Custom Wolf", new[] { "ChersoneseDungeon4" });

// Or park a prebuilt DORMANT body directly under the spawn kind (the store takes ownership and
// runs SpawnKit's own normalization pass on it) — mints then need no donor load at all.
DonorKit.TemplateStore.RegisterTemplate("My Custom Wolf", dormantClone);
```

## Settings

`BepInEx/config/cobalt.donorkit.cfg`:

| Section | Key | Default | Effect |
|---|---|---|---|
| `[Rig]` | `RepairSkinnedBones` | `true` | Repair skinned-mesh bones that reference transforms outside the harvested creature's own hierarchy. They die with the donor scene, and the creature then renders as a stretched vertical line. `false` = audit and log only, no mutation — the `[TEMPLATE] rig` line still fires either way. |
| `[Expedition]` | `CaptureOnSceneEntry` | `false` | When entering an oversized region/town that still has uncached donor species, run the capture payload in place (zero loading screens). Off by default — it is the one path that does heavy work the player never asked for. |
| `[Expedition]` | `AutoWarmAtBoot` | `needed` | Once-per-launch template warm at the first gameplay player-ready: `needed` (only when the active companion's species has no other source) / `all` (re-dump every manifest scene with uncached species) / `off`. |
| `[Expedition]` | `AlwaysWarmSpecies` | *(empty)* | Comma-separated species warmed at the same boot pass regardless of mode. Empty = the defaults consumer mods register in code (Beastwhispering registers Pearlbird); a non-empty list replaces them; `none` disables warming entirely. |
| `[Expedition]` | `AutoWarmRetrySeconds` | `60` | How long the boot warm polls for an in-flight load to settle before giving up. |
| `[Expedition]` | `ReturnRetrySeconds` | `20` | Safety net: re-request the return leg if the request never turned into a load. |
| `[Expedition]` | `AllowCoop` | `false` | Allow an expedition to start with guests connected. Off by default — a mid-trip guest gate wedge can strand the whole party in the donor region. |
| `[Expedition]` | `GuestAutoContinue` | `true` | Guest-side: auto-pass this machine's continue gates while the host's expedition is in flight (keyed on the `CK_EXPED` room property; inert in ordinary play). |

### Example configuration

```ini
[Rig]

## Repair SkinnedMeshRenderer bones that reference transforms OUTSIDE the harvested creature's own hierarchy.
# Setting type: Boolean
# Default value: true
RepairSkinnedBones = true

[Expedition]

## One-time template-cache warm at the FIRST gameplay player-ready of each launch.
# Setting type: String
# Default value: needed
AutoWarmAtBoot = needed

## Comma-separated species warmed at the same once-per-launch boot evaluation.
# Setting type: String
# Default value:
AlwaysWarmSpecies = Pearlbird
```

> **Moved in 0.1.0.** `[Rig]` and `[Expedition]` used to live in `cobalt.companionkit.cfg`. Entries
> left behind in that file are now inert and the ones here start at their defaults — the defaults are
> identical, so a stock install is unaffected; if you had customized `[Expedition]` values, re-apply
> them here. (An older *Beastwhispering* `[Expedition]` section still migrates automatically into
> this file.)

The species→scene table itself is data, not config, and ships embedded in the kit. Override or extend
it without rebuilding anything by dropping a `DonorScenes.txt` into `BepInEx/config/`:

```
# Term=Scene1,Scene2   — '#' comments and blank lines ignored; repeated keys append
Pearlbird=ChersoneseNewTerrain,CierzoNewTerrain
Hyena=Emercar_Dungeon5,ChersoneseDungeonsSmall
```

## Commands

Write a verb into `BepInEx/config/DonorKit_cmd.txt` and it runs on the next poll (even while the game
is paused). `help` or an unknown verb lists them all.

| Verb | Does |
|---|---|
| `photondump` | Photon view-registry health — the donor-view-collision diagnostic |
| `audiodump` | Audit the global audio manager for destroyed registrants |
| `audioprune` | Drop destroyed sound players from the static registry, then audit |
| `terraindump` | Snapshot terrain health now (`[TERRAIN]` mesh tiles / broken meshes / missing LOD2) |
| `terrainfix` | Force blanked terrain tiles onto the coarse LOD2 mesh. **Draws the ground at the wrong height** — a zone reload is the clean fix |
| `scenedump` | List every build scene, for donor-scene name resolution |
| `selftest` | Donor table non-empty, no harvest window leaked open |

The first five ship as a pack a consumer can register on its own channel:
[CompanionKit](./companionkit.md) does (`ck_cmd.txt`), as does Beastwhispering (`bw_cmd.txt`), so
you can reach them there without switching files. SpawnKit does not — use `DonorKit_cmd.txt`.

The expedition/template verbs (`expedition`, `templateclear`, `expeditionreset`, `templateprobe`)
deliberately stayed on the CONSUMER channels (`ck_cmd.txt`, with `bw_cmd.txt` twins) even though the
tier moved here: `templateclear`'s full reset also re-arms CompanionKit's effigy harvest gate, which
only the consumer can reach — registering a store-only twin on `DonorKit_cmd.txt` would recreate
exactly the two-verbs-drift the single entry point exists to prevent.

## For modders

Depend on DonorKit like any kit — a `Private=false` reference plus
`[BepInDependency("cobalt.donorkit")]` — then start a harvest coroutine.

```csharp
// Ask for a species by name. The kit resolves it against the donor table (exact key, else the
// longest matching term), and walks the candidate scenes in order until one yields a live creature.
if (DonorHarvest.TryGetDonorScenes("Pearlbird", out List<string> scenes, out string searchTerm))
{
    StartCoroutine(DonorHarvest.HarvestChain(scenes, searchTerm,
        // THE PAYLOAD. Runs while the donor is loaded and frozen, with the live Character in hand.
        // Must be synchronous, and must not keep references into the donor scene — clone what you
        // want out, everything else dies on the unload a moment later.
        use: src => UnityEngine.Object.Instantiate(src.gameObject, MyDormantHolder().transform),
        // Called exactly once, with whatever the payload returned — or null if the whole chain
        // came up dry (wrong scene name, no live match, payload threw).
        onResult: r =>
        {
            var body = r as GameObject;
            if (body == null) { Log.LogWarning("no Pearlbird donor available"); return; }
            // ... your mod owns it from here
        }));
}
```

One donor load can serve many species — and because cycles are the scarce resource, it should. Use
the whole-scene form when you are stocking more than one:

```csharp
// CompanionKit.Core.DonorTable.KeysForScene tells you everything this scene can donate.
StartCoroutine(DonorHarvest.HarvestScene(sceneName, $"batch={keys.Count} species",
    useScene: donor =>
    {
        foreach (string key in keys)
        {
            Character src = DonorHarvest.FindLiveInScene(donor, key);
            if (src != null) Capture(src, key);
        }
        return keys.Count;
    },
    onResult: n => Log.LogMessage($"stocked {n} species in one cycle")));
```

**API notes and traps**

- **`IdentityFor(key, donor)`, never `donor.Name`.** For a pinned table row the frozen donor reports
  its serialized base-prefab name — all three Caldera gargoyle bosses call themselves "Shell Horror".
  Recording that name means re-forming the wrong creature one reload later.
- **The chain prefers an exact name across scenes over a substring match earlier in it.** Asking for
  "Hyena" when scene 1 holds only an "Armored Hyena" defers that scene and keeps looking; if nothing
  exact turns up, it goes back for the substring donor — exactly one re-load, never a second sweep.
- **A substring match is a *body*, not an *identity*.** Asking for "Coralhorn" can legitimately hand
  you an "Alpha Coralhorn". That is a fine fallback for a cosmetic body and a bug for anything that
  claims to *be* the species. Settle identity at capture, while the live donor is in front of you.
- **Harvests are serialized.** The guard window is one static; a second harvest waits its turn. There
  is a dead-man backstop, so a wedged harvest cannot starve saves or later harvests forever.
- **Refused cases are loud, and that is on purpose.** Harvesting the scene you are standing in is
  refused (it would freeze the real one). Oversized region/town scenes are refused (loading one
  additively crashes the game). A cold miss reports `result=FAILED` rather than quietly substituting.
- **Guests do not harvest by default** in co-op. The master does the work; see the consumer kits.

## See also

- [Kits index](./README.md)
- [ForgeKit](./forgekit.md) — the dev-tooling kit DonorKit builds on (command channel, self-test)
- [CompanionKit](./companionkit.md) — consumer: puppet bodies for persistent companions
- [SpawnKit](./spawnkit.md) — consumer: dormant templates for real vanilla hostiles
- [Wiki home](../README.md)
