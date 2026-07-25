# CompanionKit — persistent creature companions for Outward mods

**CompanionKit** is a kit for Outward that gives a mod the machinery for a **persistent creature
companion** — a beast that follows the player, fights, survives zone changes and reloads, and can be
sourced from any species in the game. It is a modder's library: on its own it adds nothing a player
sees. Players get companions through the mods built on it, [Beastwhispering](../mods/beastwhispering/README.md)
(tame wild animals as pets) and [Hireling](../mods/hireling.md).

**At a glance**
- Type: reusable library (kit) — the mid-tier "companion spine"
- Requires: BepInEx 5 (Mono branch), [ForgeKit](./forgekit.md), [AggroKit](./aggrokit.md), [NetKit](./netkit.md). No SideLoader.
- Config: `BepInEx/config/cobalt.companionkit.cfg`
- Commands: `BepInEx/config/ck_cmd.txt`

## For players

You don't install or interact with CompanionKit directly. It arrives as a dependency of a mod that
uses it — install that mod (and let its dependencies come along), then follow that mod's page:
[Beastwhispering](../mods/beastwhispering/README.md) or [Hireling](../mods/hireling.md).

## How it works

Outward has no way to spawn a *fresh* creature at runtime — every creature is baked into the scene it
lives in, and the only Characters the engine can spawn and persist are humanoids and ghosts. So a
persistent beast companion can't just be summoned. CompanionKit builds one out of the parts the engine
*does* allow, in four moving pieces:

**1. The puppet (the body you see).** A live creature can be cloned with `Object.Instantiate` — its
visuals and animator copy cleanly, but its AI does not (the AI's state tree is a separate object that
isn't cloned, so a naive clone throws errors forever). CompanionKit therefore **brain-strips** the
clone: it destroys the AI, colliders, physics and networking, leaving a pure animated model, then
**puppets** it — driving movement through the creature's own `NavMeshAgent` and its animator, following
the player every frame. Colliders are off deliberately: a live-collider body with no AI would trap the
player in permanent combat. A root-bone auto-calibration step fixes creatures whose walk animation
would otherwise make the whole skeleton drift and snap.

**2. The anchor (the combat body).** Because the puppet has no colliders, enemies can't see or hit it.
So an **invisible vanilla ally** — the "anchor" — is spawned as the companion's real combat body. It
absorbs enemy detection, aggro and damage; it owns the companion's HP and its stat modifiers. The
anchor is **welded to the puppet every frame** so the two move as one, while the puppet remains the
single movement brain.

**3. Persistence.** The kit ships the body-acquisition paths and an attribute codec but **no save
format** — the consumer mod owns its own save file. Across in-session zone changes the body rides
`DontDestroyOnLoad` and is re-warped to the player on each scene load. Across a full reload the consumer
re-acquires a fresh body from its saved species + captured attributes.

**4. Body supply.** A companion needs a live creature to clone from. Three routes cover every case:

- **Nearby wild** — clone a creature standing next to the player.
- **Donor harvest** — no wild source needed. Any area is an ordinary scene that can be loaded
  *additively* in the background (~1 second), its baked creatures wake up as live instances, one is
  cloned, and the scene is unloaded. `DonorHarvest` wraps this whole recipe plus the audio-registry,
  Photon-view-collision and mid-harvest-save guards that make it survivable. A species→scene table
  ships embedded and is config-overridable.
- **Expeditions** — for creatures that live *only* out in an open-world region or town. Those scenes
  are too large to load additively (it crashes), so instead an expedition makes a **real round trip
  through the vanilla loading screen** to the region, batch-captures a dormant body *template* for
  every species that scene can donate, and returns the party to where it started. Paid once, those
  creatures are available (load-free) for the rest of the session — and the capture can be persisted
  and warmed automatically (see *Expeditions* below).

## Expeditions

Most companions are cheap to source: their donor scene loads additively on demand. Open-world regions
and towns can't be loaded that way, so a creature that lives only in a region needs an expedition — the
loading-screen round trip described above. The template it captures is cached for the session, and the
kit can persist and pre-warm that cache so a region-only headline creature is ready before a fresh
save's first reload.

Expeditions are **host/offline only** and, by default, are refused while other players are connected
(each leg ends in a per-machine "press any key" continue gate that can strand a co-op party mid-trip).

**Manifest** — `BepInEx/config/ck_expeditions.txt` records every scene whose capture completed. Safe to
edit or delete.

## Settings

`BepInEx/config/cobalt.companionkit.cfg`. Edit while the game is closed, or edit and run `ckreload`
(BepInEx has no config file-watcher).

### `[Expedition]`

| Key | Default | Effect |
|---|---|---|
| `CaptureOnSceneEntry` | `false` | When on, entering an oversized region that still has uncached donor species runs the capture in place — zero loading screens, but it clones every uncached species on *every* such entry, so it's off by default. |
| `AutoWarmAtBoot` | `needed` | Once-per-launch template warm at the first gameplay ready. `needed` = only when the active companion's species is region-only and has no cached body; `all` = re-warm every recorded scene with uncached species; `off` = never. |
| `AlwaysWarmSpecies` | `Pearlbird` | Comma-separated species warmed at the same boot pass regardless of `AutoWarmAtBoot`'s mode (the packaging knob for a mod whose headline creature is region-only — a hand-off bundle can't ship a `.cfg`, so this ships as a code default). Only fires on a launch with no companion, or when the active companion's species is itself listed — a save bonded to another species never pays a boot trip it didn't ask for. Set empty to opt out. |
| `AutoWarmRetrySeconds` | `60` | If a vanilla load is still in flight at boot, how long the auto-warm waits for the loader to settle before giving up. `0` = don't wait. |
| `ReturnRetrySeconds` | `20` | Safety net for an expedition's return leg: re-request the trip home if no load has started this long after asking. `0` = never retry (the watchdog still applies). |
| `AllowCoop` | `false` | Allow an expedition to start while guests are connected. Off by design — a stuck guest continue-gate can wedge the whole party. |
| `GuestAutoContinue` | `true` | Guest-side safety: while a host-run expedition is in flight, auto-pass this client's continue gate. Inert in ordinary play. |

### `[Rig]`

| Key | Default | Effect |
|---|---|---|
| `RepairSkinnedBones` | `true` | Repair mesh bones that reference transforms outside the harvested creature's own hierarchy (they die with the donor scene and the body renders as a stretched line). `false` = log only. |

### `[Effigy]` — co-op cosmetic bodies

Effigies show *other players'* companions on this machine as local, purely cosmetic bodies that follow
each pet's networked anchor. They have no combat, HUD or simulation.

Cross-machine companion bodies are the newest part of the kit and have limitations a player will see. An
effigy needs a live creature to clone into its body: a species that isn't already cached on this machine
and has no matching wild nearby appears as a neutral ghost stand-in that idles in place until a source
creature comes within range, at which point it forms into the real body. The same fallback applies to a
guest's own companion when it re-forms with no creature nearby to clone.

| Key | Default | Effect |
|---|---|---|
| `EnableCompanionEffigies` | `true` | Show other players' companions as local cosmetic bodies. Off despawns them on the next tick (identities are kept — flipping back on resumes with no rejoin). |
| `MaxBodies` | `4` | Upper bound on simultaneous effigy bodies (each is a full brain-stripped clone). Extra rows wait for a slot. |
| `EnableEffigyHarvest` | `true` | When an effigy's species isn't cached and no wild is nearby, let the master run the same additive donor harvest a local pet would — paid once per species. Off = wild → cache → ghost only. |
| `HarvestRetryMinutes` | `10` | After an effigy harvest comes up dry, how long before that species may retry. Floor 10 seconds. |
| `PinToAnchor` | `false` | Pin each effigy body onto its pet's networked anchor replica instead of navigating it independently. |
| `AnchorAnimSpy` | `false` | Diagnostic: periodically dump every foreign anchor replica's animator state. |

### Example configuration

`BepInEx/config/cobalt.companionkit.cfg` — created on first launch. Excerpt:

```ini
[Expedition]
CaptureOnSceneEntry = false
AutoWarmAtBoot = needed
AlwaysWarmSpecies = Pearlbird

[Rig]
RepairSkinnedBones = true

[Effigy]
EnableCompanionEffigies = true
MaxBodies = 4
```

Dev commands run from `BepInEx/config/ck_cmd.txt`, and the species→scene donor table is
config-overridable by dropping `BepInEx/config/DonorScenes.txt` (see *How it works*). The
shared-settings overlay is `config/shared/cobalt.companionkit.cfg.overlay` (see `config/README.md`).

## Commands

Write a verb into `BepInEx/config/ck_cmd.txt`; it runs on the next poll (even while paused). An unknown
verb or `help` lists them all.

| Verb | What it does |
|---|---|
| `expedition` | No args: status — harvest state, template-cache census, manifest, config, this launch's auto-warm decision. |
| `expedition <scene\|species>` | Round trip to an oversized donor scene, caching a body template for every species it donates. Hands-off (loading-screen gates pass automatically). Species with an ordinary additive donor are refused toward that cheaper path. Host/offline only. |
| `expeditionreset` | Force the expedition guard open after a wedge (doesn't teleport anyone — use `goto` for that). |
| `templateclear` | Free the session body-template cache (logs the count freed). |
| `templateprobe` | Per cached template: species + a defensive mesh-readability read. |
| `photondump` | Photon view-registry health (donor-harvest diagnostics). |
| `audiodump` / `audioprune` | Audit / prune the global audio registrant list (donor scenes self-register audio that must be detached on unload). |
| `terraindump` / `terrainfix` | Snapshot / repair terrain-tile render holes (a failed harvest can blank a LOD tile; a zone reload is the clean fix). |
| `proxydump` / `proxykill <ownerUid>` | Co-op guest-pet proxy census / tear down one proxy row. |
| `effigydump` | Co-op effigy census on this machine. |
| `netbusdump` | Co-op network census (delegates to [NetKit](./netkit.md)'s `netdump`). |
| `ckreload` | Re-read `cobalt.companionkit.cfg` from disk. |

## For modders

CompanionKit gives you every path to acquire a companion body and drive it; **you own persistence** (the
save format) and your player-facing features (skills, feeding, UI).

Reference both DLLs from the kit's own folders — never copy them into yours:

```xml
<ProjectReference Include="..\CompanionKit\CompanionKit.csproj" Private="false" />
<ProjectReference Include="..\..\core\CompanionKit.Core\CompanionKit.Core.csproj" Private="false" />
```

```csharp
[BepInPlugin(GUID, NAME, VERSION)]
[BepInDependency(CompanionKit.Plugin.GUID, BepInDependency.DependencyFlags.HardDependency)]
public class Plugin : BaseUnityPlugin
{
    void Awake()
    {
        CompanionRuntime.Log = Logger;
        CompanionRuntime.DefaultSettings = new MySettings(); // ICompanionSettings over YOUR config
    }
}
```

Lifecycle — `Companion` is the aggregate whose lifetime is the *bond*, not the body (bodies get
destroyed and rebuilt across zones and reloads; standing orders and anchor state must not):

```csharp
var companion = new Companion();
companion.Anchor.OnAnchorDeath          += () => { /* your downed policy */ };
companion.Anchor.OnAnchorCriticallyHurt += () => { /* warn the player */ };

// acquire a body (any route):
CompanionBody body = BodyFactory.BuildPuppet(src, owner, consume: true);   // the wild source is consumed
CompanionBody body = BodyFactory.BuildPuppet(src, owner, consume: false);  // the wild source survives
// or, with no wild source present:
//   yield return DonorHarvest.HarvestChain(scenes, species, owner, b => body = b, ...);

// weld the anchor to the body and (optionally) enable combat:
CompanionCombat combat = companion.AdoptBody(body, enableCombat: true);

// push captured stats (idempotent — re-push any time; null clears back to defaults):
companion.Anchor.ApplyVitals(maxHealth);
companion.Anchor.ApplyCreatureStats(effectiveAttributes);
combat.SetAttackProfile(effectiveAttributes);

// standing orders (survive body re-forms because they live on the bond):
companion.Stance.CommandEngage(target);   // priority 0, honored at any range
companion.Stance.CommandDisengage();       // passive + a run-home grace window
```

Key public surface:

| Type | Role |
|---|---|
| `Companion` | The bond aggregate: `Body`, `Anchor`, `Stance`, `Combat`, and `AdoptBody(body, enableCombat)` which welds and wires them. |
| `BodyFactory.BuildPuppet(src, player, consume, …)` | Clone + brain-strip a live creature into a `CompanionBody`. Consume mode replaces the wild one; no-consume leaves it alive. |
| `CompanionBody` | The puppet: NavMeshAgent follow, zone re-placement, leash/warp, animator driving. Exposes `OnAfterMove` so subscribers run after the final pose is written. |
| `CompanionAnchor` | The invisible combat body: HP, stat seams (`ApplyVitals`, `ApplyCreatureStats`), the `OnAnchorDeath` / `OnAnchorCriticallyHurt` events. |
| `CompanionCombat` | Manual combat for a brain-stripped body: target selection (commanded > defend-the-anchor > assist-the-owner), typed damage profiles, and the public `DealDamage` / `DealDamageTyped` seam for consumer special attacks. Set profiles with `SetAttackProfile`. |
| `AttributeCapture.From(character)` | Read a live `Character`'s real stats (resistances, health, natural weapon damage, movement) into a portable `Core.CreatureAttributes`. |
| `DonorHarvest` | "Body anywhere": additively load any donor scene, clone, unload, with all the guards. |
| `ExpeditionHarvest` / `BodyTemplateCache` / `ExpeditionOrchestrator` | Region-only creatures via a loading-screen round trip, plus the session template cache and the boot warm/persistence policy. |

`CompanionKit.Core` is a game-reference-free, unit-tested compute layer: weld policy, target selection,
the attribute codec (`CreatureAttributes.Flatten` / `Parse` is one save-safe string), the donor table,
and the expedition manifest and warm policy.

**Consumer seams for the expedition tier:** set `ExpeditionOrchestrator.AutoWarmDeferred = true` in your
`Awake` if you want to fire the boot warm yourself, after your own saved companion state is loaded, so
`needed`-mode warm knows which species is active.

### Scope and caveats

- The API is instance-based, but **one companion per owner** is the well-trodden path. Several at once
  should work but is unexercised (the vanilla summon slot is one-per-player, and multi-anchor aggro is
  untested).
- The kit ships the attribute codec and every body-acquisition path but **no save file** — you own your
  own format.
- Special attacks stay consumer-side; `CompanionCombat.DealDamage` is the public seam.
- **Beast bodies are the primary path.** Humanoid bodies use a different ghost stand-in route that has
  had far less mileage.

## Compatibility

Outward must be on its **Mono** Steam beta branch, not the default IL2CPP build. If the game runs but no
BepInEx mods load and there's no crash log, this is almost always why (Steam → game Properties → Betas →
select `mono`).

## See also

- [Kits index](./README.md) · [Wiki home](../README.md)
- Dependencies: [ForgeKit](./forgekit.md), [AggroKit](./aggrokit.md), [NetKit](./netkit.md)
- Built on top: [SpawnKit](./spawnkit.md)
- Mods that use it: [Beastwhispering](../mods/beastwhispering/README.md), [Hireling](../mods/hireling.md)
