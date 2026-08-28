# CompanionKit — persistent creature companions for Outward mods

**CompanionKit** is a kit for Outward that gives a mod the machinery for a **persistent creature
companion** — a beast that follows the player, fights, survives zone changes and reloads, and can be
sourced from any species in the game. It is a modder's library: on its own it adds nothing a player
sees. Players get companions through the mods built on it, [Beastwhispering](../mods/beastwhispering/README.md)
(tame wild animals as pets) and [Hireling](../mods/hireling.md).

**At a glance**
- Type: reusable library (kit) — the mid-tier "companion spine"
- Requires: BepInEx 5 (Mono branch), [ForgeKit](./forgekit.md), [DonorKit](./donorkit.md), [AggroKit](./aggrokit.md), [NetKit](./netkit.md). No SideLoader.
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
  cloned, and the scene is unloaded. This is **[DonorKit](./donorkit.md)**'s engine: `DonorHarvest`
  wraps the whole recipe plus the audio-registry, Photon-view-collision and mid-harvest-save guards
  that make it survivable, and a species→scene table ships embedded there and is config-overridable.
  It is its own kit so SpawnKit doesn't have to depend on the pet library for creature
  supply; CompanionKit still drives it, and `CompanionKit.DonorHarvest` remains as a forwarder.
- **Expeditions** — for creatures that live *only* out in an open-world region or town. Those scenes
  are too large to load additively (it crashes), so instead an expedition makes a **real round trip
  through the vanilla loading screen** to the region, batch-captures a dormant body *template* for
  every species that scene can donate, and returns the party to where it started. Paid once, those
  creatures are available (load-free) for the rest of the session. The whole tier — the trip state
  machine, the template store, the boot warm and its `[Expedition]` config — is
  **[DonorKit](./donorkit.md)'s**; CompanionKit still drives it (a bodiless pet
  re-forms from the store in milliseconds) and the `expedition`/`templateclear` verbs still answer
  on this kit's `ck_cmd.txt`.

## Settings

`BepInEx/config/cobalt.companionkit.cfg`. Edit while the game is closed, or edit and run `ckreload`
(BepInEx has no config file-watcher).

> **Moved out:** the `[Expedition]` and `[Rig]` sections now live in
> `BepInEx/config/cobalt.donorkit.cfg` — see [DonorKit's settings](./donorkit.md#settings) for the
> tables and an example. Entries left behind in this kit's cfg are inert; the defaults are
> identical, and hand-tuned `[Expedition]` values should be re-applied in DonorKit's file. The
> expedition manifest keeps its historical name (`ck_expeditions.txt`) so existing installs keep
> their record.

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
| `PinLerp` | `1` | Position follow factor per frame while pinned. `1` = weld straight onto the anchor (which the engine already smooths — the expected setting); lower only if a session shows visible stepping. Clamped `0.05`–`1`. |
| `PinCatchUpSpeed` | `12` | Metres/second a pinned body may close a gap larger than ~1.5 m at, so it runs back after an anchor-missing window instead of teleporting. Clamped `1`–`50`. |
| `PinSnapDistance` | `12` | Gap (metres) at or past which a pinned body teleports onto its anchor instead of gliding — zone warps and respawns must not send a body running across the map. Clamped `5`–`100`. |
| `AnchorNetLerpSpeed` | `4` | Re-tunes the game's own network smoothing on *foreign* anchor replicas here (vanilla `1`), so a pinned body doesn't wear the replica's lag on screen. `0` leaves vanilla untouched. Clamped `0`–`20`. |
| `AnchorNetMoveSpeed` | `1` | The companion setting to the above, re-tuning the replica's move speed (vanilla `0.2`). `0` leaves vanilla untouched. Clamped `0`–`10`. |
| `AnchorAnimSpy` | `false` | Diagnostic: periodically dump every foreign anchor replica's animator state. |

### `[Diag]` — the orphan-body reaper

A companion body that outlives its bond (a confirmed leak class) used to keep following the player
as an unreachable duplicate. The kit now tracks every live body and destroys any that nothing owns,
after a generous grace window.

| Key | Default | Effect |
|---|---|---|
| `OrphanBodyPolicy` | `Reap` | `Reap` = warn, then destroy an unowned body; `LogOnly` = warn only (the live escape hatch if you suspect a false positive); `Off` = no sweep. |
| `OrphanBodyGraceSeconds` | `90` | Never judge a body younger than this (in-flight builds legitimately go unowned for a few seconds). |
| `OrphanBodyCondemnSeconds` | `20` | How long an orphan is warned about before it is destroyed. |

### Combat style — `Chase` vs `Opposite` (host-configured)

Every companion body carries a `CombatStyle`. `Chase` (the default) closes to melee on its target.
`Opposite` stations the body on the **far side** of the target — so the enemy ends up between the
owner and the companion — and plants there; Beastwhispering uses it for the Pearlbird so an archer
keeps a clear line of fire. A body keeps its station while it is out of the line between the
owner and the enemy (seen from the owner) or the enemy has barely moved, and once the fight has
settled on the pet only the line test applies; if the enemy runs off toward the owner (the *far*
rule) it re-stations regardless. Re-stations are rate-limited, and after `StationMaxRestations`
in one fight the body simply chases for the rest of it, so it can never dance around a mob.

The style is chosen by the host mod per species. The tuning knobs are the host's too — for
Beastwhispering they live in `BepInEx/config/cobalt.beastwhispering.cfg`, section `[Combat]`:

| Key | Default | Effect |
|---|---|---|
| `StationRingFraction` | `0.6` | Where the station sits: this fraction of the pet's attack range past the enemy, on the owner→enemy line. |
| `StationLineAngleDeg` | `20` | Seen from the owner, the pet counts as "in the corridor" (between owner and enemy) when the enemy↔pet angle is under this — the station is then re-picked. |
| `StationRestationMeters` | `3` | While the pet is in the corridor, the enemy must also have moved at least this far since the station was set (ignored once the fight has settled on the pet). |
| `StationRestationSeconds` | `2` | Minimum seconds between re-stations. |
| `StationArriveMeters` | `0.8` | Within this distance of the station the body counts as planted; a re-pick this close to where the pet already stands is skipped. |
| `StationMaxRestations` | `6` | Re-stations allowed per engagement; past it the body chases (logged once). |
| `StationFarMeters` | `3` | An enemy farther than attack range + this from the pet forces a re-station regardless of the line test. |
| `StationProgressMeters` | `1` | Converge or chase: a "the enemy left me" re-station must close at least this much ground on the new station. Two in a row that do not means the pet gives up the stance and plainly chases for the rest of that fight. |
| `StationEnemyFastMetersPerSecond` | `0` | Fast hold: while the enemy is moving faster than this (m/s) the "enemy left me" re-station waits — a running mob drags the station away as fast as the pet walks, so the pet keeps the station it has and the host's converge pin brings the mob to it. `0` = off (identical to before). A walking mob is ~1–2 m/s, a charging one 6+; try `4`. |

```ini
[Combat]
StationRingFraction = 0.6
StationLineAngleDeg = 20
StationRestationMeters = 3
StationRestationSeconds = 2
StationArriveMeters = 0.8
StationMaxRestations = 6
StationFarMeters = 3
StationProgressMeters = 1
StationEnemyFastMetersPerSecond = 0
```

The enemy's speed is measured by the body from consecutive positions (`CompanionKit.Core.SpeedEstimate`:
0.1 s windows, blended, a >30 m/s jump is a teleport and ignored). The hold applies only to the
"enemy left me" rule (and a planted pet's re-close against that same far, running enemy); the
line-of-fire re-station is never held. `stationdump` shows the held windows as `fastHolds=`.

(These are the defaults.) The log tag is `[STATION]` — `set: why=first` on the first station,
`restation #n why=…` on each later one, `planted` on arrival, `cap fallback` once past the cap;
Beastwhispering's `stationdump` prints the live state.

### Example configuration

`BepInEx/config/cobalt.companionkit.cfg` — created on first launch. Excerpt:

```ini
[Effigy]
EnableCompanionEffigies = true
MaxBodies = 4
```

Dev commands run from `BepInEx/config/ck_cmd.txt`, and the species→scene donor table is
config-overridable by dropping `BepInEx/config/DonorScenes.txt` (see *How it works*). The
expedition/rig settings live in [DonorKit's cfg](./donorkit.md#settings), and its shared-settings
overlay is `config/shared/cobalt.donorkit.cfg.overlay` (see `config/README.md`).

## Commands

Write a verb into `BepInEx/config/ck_cmd.txt`; it runs on the next poll (even while paused). An unknown
verb or `help` lists them all.

| Verb | What it does |
|---|---|
| `expedition` | No args: status — harvest state, template-cache census, manifest, config, this launch's auto-warm decision. |
| `expedition <scene\|species>` | Round trip to an oversized donor scene, caching a body template for every species it donates. Hands-off (loading-screen gates pass automatically). Species with an ordinary additive donor are refused toward that cheaper path. Host/offline only. |
| `expeditionreset` | Force the expedition guard open after a wedge (doesn't teleport anyone — use `goto` for that). |
| `templateclear [all]` | Free the session body-template cache. Default keeps `origin=prebuilt` (bundle) templates and logs `cleared N harvested, kept M prebuilt`; `all` wipes everything. |
| `templateprobe` | Per cached template: species + a defensive mesh-readability read. |
| `photondump` | Photon view-registry health (donor-harvest diagnostics). |
| `audiodump` / `audioprune` | Audit / prune the global audio registrant list (donor scenes self-register audio that must be detached on unload). |
| `terraindump` / `terrainfix` | Snapshot / repair terrain-tile render holes (a failed harvest can blank a LOD tile; a zone reload is the clean fix). |
| `proxydump` / `proxykill <ownerUid>` | Co-op guest-pet proxy census / tear down one proxy row. |
| `effigydump` | Co-op effigy census on this machine. |
| `effigyrebind <ownerUid\|all>` | Reset an effigy row's local binding so the next reconcile re-resolves from scratch. |
| `bodycensus` | Every live companion body on this machine — id, species, origin, age, ownership, position and the pathfinding agent's internal position. |
| `netbusdump` | Co-op network census (delegates to [NetKit](./netkit.md)'s `netdump`). |
| `ckreload` | Re-read `cobalt.companionkit.cfg` from disk. |

`photondump`, `audiodump`, `audioprune`, `terraindump` and `terrainfix` are
[DonorKit](./donorkit.md)'s shared donor-diagnostic pack, registered here as well so you can reach
them without switching channels; they also answer on `DonorKit_cmd.txt`. CompanionKit does **not**
register ForgeKit's shared `CommonVerbs` pack — `give`, `teleport`, `reloadcfg` and the rest are not
available on `ck_cmd.txt`.

## For modders

CompanionKit gives you every path to acquire a companion body and drive it, plus
the whole **persistence lifecycle** (when to save, when to re-form, the body-acquisition ladder — see
*Persistence* below); **you own the state** (what a save row means, its codec/schema) and your
player-facing features (skills, feeding, UI).

Reference both DLLs from the kit's own folders — never copy them into yours:

```xml
<ProjectReference Include="..\CompanionKit\CompanionKit.csproj" Private="false" />
<ProjectReference Include="..\..\core\CompanionKit.Core\CompanionKit.Core.csproj" Private="false" />
```

```csharp
[BepInPlugin(GUID, NAME, VERSION)]
[BepInDependency(CompanionKit.Plugin.GUID, CompanionKit.Plugin.VERSION)]   // VERSION = the min-version floor, see kits/versioning.md
public class Plugin : BaseUnityPlugin
{
    CompanionHost _host;

    void Awake()
    {
        // ONE host handle per consumer — it carries your logger + your
        // ICompanionSettings and derives your log-tag suffix from the GUID automatically
        // ("cobalt.hireling" → "[ANCHOR/HIRE]" etc.; pass tagSuffix: "" for plain tags,
        // combatTag: "…" to rename the combat component's log noun — default "COMBAT").
        // The old CompanionRuntime.Log / DefaultSettings globals are obsolete shims:
        // they had no ownership rule, so with two consumers installed BepInEx load order decided
        // whose logger got all kit output. Don't set them.
        _host = CompanionHost.Create(GUID, NAME, Logger, new MySettings());
    }
}
```

Lifecycle — `Companion` is the aggregate whose lifetime is the *bond*, not the body (bodies get
destroyed and rebuilt across zones and reloads; standing orders and anchor state must not):

```csharp
var companion = new Companion(_host);   // settings ride the host; a per-companion override is the 2nd arg
companion.Anchor.OnAnchorDeath          += () => { /* your downed policy */ };
companion.Anchor.OnAnchorCriticallyHurt += () => { /* warn the player */ };
// (wire these BEFORE AdoptBody, or accept the one-time "[COMPANION] note: no anchor consequence
//  events wired" line — a downed companion silently respawns after AnchorRespawnSeconds otherwise)

// acquire a body (any route):
CompanionBody body = BodyFactory.BuildPuppet(src, owner, consume: true);   // the wild source is consumed
CompanionBody body = BodyFactory.BuildPuppet(src, owner, consume: false);  // the wild source survives
// or, with no wild source present:
//   yield return DonorHarvest.HarvestChain(scenes, species, owner, b => body = b, ...);

// weld the anchor to the body and (optionally) enable combat; humanoids may ask for the
// draw/sheathe flourish here (it is an AdoptBody parameter, not a field to poke):
CompanionCombat combat = companion.AdoptBody(body, enableCombat: true, weaponPosture: false);

// push captured stats (idempotent — re-push any time; null clears back to defaults):
companion.Anchor.ApplyVitals(maxHealth);
companion.Anchor.ApplyCreatureStats(effectiveAttributes);
combat.SetAttackProfile(effectiveAttributes);

// standing orders (survive body re-forms because they live on the bond):
companion.Stance.CommandEngage(target);   // priority 0, honored at any range
companion.Stance.CommandDisengage();       // passive + a run-home grace window
```

### The combat target ladder

Every 0.3 s an engaged companion re-picks what to fight. The rules are pure
(`Core.CombatTargetPolicy.Decide`) — the game side only gathers the facts:

| Prio | Source | Gate |
|---|---|---|
| 0 | **commanded** — your explicit `CommandEngage` order | no range gate at all; only the combat leash |
| 1 | **owner-focus** — the enemy the owner's own attack last landed on | `OwnerFocusRange` (60 m), held 8 s, refreshed by every landed hit |
| 2 | **anchor-defend** — whoever the companion's own anchor has locked (i.e. is attacking it) | `AggroRange` (12 m) |
| 3 | **player-engaged** — the nearest AI in `player.EngagedCharacters` | `AggroRange`, and the player must be in combat |

Any pick beyond `CombatLeashDistance` is dropped as stale, and a stale *commanded* target is also
un-commanded so the next scan cannot re-pick it. Each transition is logged with its reason:
`[<COMBATNOUN>] target: none -> Bandit (owner-focus, petPos=…)`.

**Owner-focus** (2026-08-24) is the tier that makes a companion useful to a ranged player. Both of
the older auto tiers are gated on `AggroRange`, which is measured *companion*→enemy; and vanilla
does not even list a target you shot at 30 m in `player.EngagedCharacters` until it aggros back and
closes to 25 m (`Character.UpdateCombatStatus` / `CheckIfCombatWorthy`). So a bow user's companion
simply waited. `OwnerFocusTracker` supplies the missing signal by observing the owner's own landed
hits — `Character.HasHit` for melee and `Projectile.OnProjectileHit` for ranged (the melee seam
excludes bows, so both are needed) — and it is deliberately placed *above* self-defence: the
companion will leave whatever is biting it to go fight what its owner is attacking. Consumers turn
it off with `ICompanionSettings.AssistOnOwnerHit`, which restores the previous ladder exactly.

In multiplayer nothing new crosses the wire: the hit callbacks run on the attacking player's own
machine, so a guest's own shot is a local event and the resulting engage reaches the master through
the existing `ck.proxy.target` mandate.

Key public surface:

| Type | Role |
|---|---|
| `CompanionHost` | The per-consumer handle (create once in `Awake`): your logger + settings, an auto-derived log-tag suffix, the combat log noun. Everything the retired `CompanionRuntime` globals used to carry, with an owner. |
| `Companion` | The bond aggregate: `Body`, `Anchor`, `Stance`, `Combat`, `AdoptBody(body, enableCombat, weaponPosture)` which welds and wires them, `Tick` (anchor upkeep **including the bodiless branch** — `ICompanionSettings.BodilessAnchor` decides destroy-vs-upkeep while no body exists), `Despawn()` (the **whole** teardown: anchor + body, stance reset), and `BindCapturedStats(get, set)` to route the capture slot through your save model. |
| `CompanionTicker` | The shared sim driver: the 2 s cadence + the per-tick order contract (`Companion.Tick`, then your `ApplyAttributes` fan-out). `Due`/`Advance()` for a consumer with its own sim, `Run(...)` for one without; `Prime()`/`Poke()` are the reset idioms. |
| `CompanionCandidate` | `IsBeast` / `IsHumanoid` — the engine's `UseLegacyVisual` discriminator, so tamable/recruitable predicates stop re-deriving it. |
| `BodyFactory.BuildPuppet(src, player, consume, …)` | Clone + brain-strip a live creature into a `CompanionBody`. Consume mode replaces the wild one; no-consume leaves it alive. |
| `CompanionBody` | The puppet: NavMeshAgent follow, zone re-placement, leash/warp, animator driving. Exposes `OnAfterMove` so subscribers run after the final pose is written. |
| `CompanionAnchor` | The invisible combat body: HP, stat seams (`ApplyVitals`, `ApplyCreatureStats`), the `OnAnchorDeath` / `OnAnchorCriticallyHurt` events. |
| `CompanionCombat` | Manual combat for a brain-stripped body: target selection (commanded > defend-the-anchor > assist-the-owner), typed damage profiles, and the public `DealDamage` / `DealDamageTyped` seam for consumer special attacks. Set profiles with `SetAttackProfile`. |
| `AttributeCapture.From(character[, host])` | Read a live `Character`'s real stats (resistances, health, natural weapon damage, movement) into a portable `Core.CreatureAttributes`. Pass your `CompanionHost` (also accepted as the optional trailing `host:` on `BuildPuppet`/`BuildHumanoidPuppet`/`BodyTemplateCache.Capture`) so the `[STATS]` capture line attributes to YOUR logger; null = the kit's. |
| `DonorHarvest` | "Body anywhere": additively load any donor scene, clone, unload, with all the guards. **Now [DonorKit](./donorkit.md)'s**; the `CompanionKit.DonorHarvest` name is a forwarder, plus the two puppet-shaped overloads that genuinely belong to this kit. |
| `ExpeditionHarvest` / `BodyTemplateCache` / `ExpeditionOrchestrator` | Region-only creatures via a loading-screen round trip, plus the session template cache and the boot warm/persistence policy. **Now [DonorKit](./donorkit.md)'s** (the store is `DonorKit.BodyTemplateStore`); these names are forwarders, except `BodyTemplateCache.PuppetFrom`/`ClearAllAndForgetMisses`, which are genuinely this kit's (the puppet build and the effigy-gate reset). |

**Co-op (the `ck` NetKit channel).** `NetBus` is a thin facade over NetKit's `ck` channel; consumers
register verb handlers and send with it rather than touching Photon. Two notes worth knowing:

- `NetBus.Register(verb, handler, HandlerRole …)` declares the role constraint (master-only,
  sender-must-be-master, no self-echo) instead of hand-writing the guard at the top of the handler —
  a violation becomes a uniform `role:*` drop in `netdump`, and a FORGOTTEN guard becomes impossible.
- `NetBus.SendToActor(actorId, verb, ownerUid, payload)` sends to one peer **by actor id**. Prefer it
  over `SendToPlayer` whenever you already hold an id (a proxy row's owner, a peer-ready flush
  target): it drops the `PhotonPlayer` round trip, and an actor that has left is NetKit's uniform
  `actor-gone` drop rather than a consumer-side warning.
- `NetBus.RegisterStore(name, StoreOptions)` registers a NetKit ReplicatedRecord store on the `ck`
  channel. Two lifecycles ride stores, wire bytes byte-identical to the pre-store
  builds in both cases (old and new builds interoperate):
  - the **effigy identity lifecycle** (`effigy`, `StoreAuthority.Owner`, the shipped
    `ck.effigy.set`/`ck.effigy.clear` verbs). The store owns the announce latch (change / ~30 s
    periodic), the late-join replay, room-change and peer-lost teardown, and the role guards;
    `Core.EffigyLedger` keeps only what a row *means* (species, tier, stance, anchor/body state).
    The stance mirror stays a separate `StateMirror` beside it.
  - the **guest-pet proxy lifecycle** (`proxy`, `StoreAuthority.Master`, the shipped
    `ck.proxy.announce`/`ck.proxy.release` verbs). The store owns
    sender↔owner binding (`ProxyPets`' ownership rungs became its `ResolveUidOwner` callback),
    the reconnect rebind rule, the 45 s owner-missing backstop, disconnect/room teardown and
    release authorization; `ProxyPets` keeps only what a row means — a **bodiless `Companion`
    aggregate** (policy `BodilessAnchor=UpkeepAnchor`) whose anchor follows the guest's replica,
    the stats/combat/stance verbs (still hand verbs, authorized against the store row's bound
    actor), and the M→G event surface. The non-record verbs' guest half is the **injected
    `ICompanionNetMirror` seam** (below).

**`ck.proxy.status` (2026-08-18).** A master→guest leg alongside the lifecycle verbs above: the
master streams the proxied anchor's live status `IdentifierName`s to that row's owner on the upkeep
tick, change-latched (an unchanged list is not resent). It exists because a guest's own pet is a
proxy row whose anchor lives on the master, so the guest could not see what its pet was suffering —
Beastwhispering's damage-over-time auras read this mirror where the master path reads the live
anchor. A master that does not send it leaves the guest snapshot-less rather than wrong, and the
consumer is expected to say so out loud rather than render a guess.

**Player-body cosmetic leg (`CompanionPlayerFx`, 0.4.8).** The transient-only twin of the pet
flourish: `CompanionPlayerFx.Flourish(ownerUid, spellKey)` plays a one-shot FX on the owning
*player's* Character — on every machine, since player replicas exist everywhere — and rides
`ck.playerfx.cast` (master → others) / `ck.proxy.playerfx.cast` (guest → master, authorized by the
owner-binding rung rather than a pet row, so a petless guest can still flash). The binder is the
target player's own Character (self-play), so a weapon-bound vanilla VFX binds to that player's
equipped weapon identically on every box. The consumer installs `CompanionPlayerFx.Resolver`
(`Func<PetFxRequest, FxRecipe>`, the same type as `CompanionPetFx.Resolver`) as the allowlist: an
unknown key is a `no-recipe` drop, never an attach. No store, no late-join replay — a lost flash is
one missed flourish. The master drops a guest report whose uid resolves no replica yet as
`no-replica` (mid-join race) and a failed binding rung as `not-owner`.

*Self-play constraint for Resolver authors:* the one-shot runs vanilla binding on the target
player's own Character (`BodyFx.cs` `PlayOneShot`, the `selfPlay` branch) with **no** T3 strip and
**no** pin — so a recipe whose prefab carries `VFXParticlesOnVisuals` (reparents a child onto the
character skeleton, where it outlives the timed Destroy) or `VFXPositionOnChar` must not be
blessed by the Resolver.

*Unarmed / unresolvable weapon slot (0.4.11):* self-play lets vanilla bind first, but
`VFXParticlesOnWeapon.ChooseRenderer` returns **null** with no weapon equipped and
`RefreshVFXOnChar` only re-stamps the ShapeModule for a SkinnedMeshRenderer or MeshRenderer — so a
mesh-shape emitter is left with no renderer and emits **nothing**. (The older "an unarmed player
gets the prefab's default shape at the clone root" note was wrong.) After Play, `PlayOneShot`
therefore rebinds **only** the emitters still holding a null target renderer onto the player's own
body mesh; a resolved weapon renderer is never touched. `BodyFx.DescribeBindings` reports what each
emitter actually bound to (`<type>@<node> -> renderer=<name|NONE>`) on the `[PLAYERFX] played …`
line. The only blessed prefab today, `VFXCounter` (VFXSystem +
VFXParticlesOnWeapon children), is clean.

### Adding a replicated cosmetic: transient vs persistent

Every replicated visual in the kit is one of two shapes. Pick by asking one question: *if a late
joiner missed the message, should they still see it?*

| | **Transient** (a moment) | **Persistent** (a state) |
|---|---|---|
| Examples | cast flourish (`ck.petfx.cast`), player flash (`ck.playerfx.cast`), effigy swing (`ck.effigy.swing`) | ward glow / lantern halo (`petfx` store rows), auras (`CompanionAura`), effigy identity |
| Lost message means | one missed animation | a desync — so it is latched, refreshed and replayed |
| Late join | nothing to replay | flushed on the peer's scene-ready edge |
| Kit primitive | `NetBus.RegisterFireAndForget` → `FireAndForgetChannel` | `NetBus.RegisterStore` → `ReplicatedStore` (or an existing store's row) |
| What you write | two verb names, a payload codec, an `apply`, an authorizer | a record key + payload codec, `OnSet`/`OnCleared` handlers, a ledger of what a row *means* |

**Transient — one registration.** `NetBus.RegisterFireAndForget(castVerb, proxyVerb, authorize,
apply, enabled)` registers both halves of the relay and hands back the one send seam:

```csharp
// boot (Init)
_cast = NetBus.RegisterFireAndForget(
    "ck.mything.cast",               // master → Others (SenderMustBeMaster | SenderMustNotBeSelf)
    "ck.proxy.mything.cast",         // guest → master (RunOnMasterOnly)
    ProxyPets.AuthorizePetOwner,     // null = ok, else the drop reason (`no-row` / `wrong-sender`);
                                     //   AuthorizePlayerOwner for a player-bound verb (`no-replica` / `not-owner`)
    ApplyWire,                       // (uid, payload, verb, out relay) — parse, then apply on the local body
    () => Enabled);                  // optional: every leg goes silent while false

// the consumer's send
_cast.Fire(ownerUid, MyCodec.Build(...));   // applies here; in a room, relays (master) or reports (guest)

// the apply seam — parsing is yours, so the drop is counted under the leg that carried it.
// Return the PARSE verdict: false means the master will not relay the bytes it just refused.
// relayPayload is the canonical rebuild the master relays (null = the received bytes verbatim).
static bool ApplyWire(string uid, string payload, string verb, out string relayPayload)
{
    relayPayload = null;
    if (!MyCodec.TryParse(payload, out var thing)) { NetBus.CountDrop(verb, "unparseable"); return false; }
    relayPayload = MyCodec.Build(thing);
    ApplyLocal(uid, thing);          // your `no-body` / `no-recipe` / `play-failed` drops live here
    return true;                     // a body/recipe miss is still true — the bytes were sound
}
```

The channel owns what the three hand-written carriers used to repeat: the in-room check, the
master-relay / guest-report fork, the `empty-identity` drop, and the rule that the owning guest's
OWN cast coming back from the master is a *skip, not a drop* (it already played in `Fire`;
counting it would make a healthy session read as steady loss). The decision ladder is pure
(`NetKit.Core.FireAndForgetLadder`, covered by `tests/NetKit.Tests`), so a new carrier inherits
tested routing and only its codec and apply are new code. `CompanionPetFx`'s cast half and
`CompanionPlayerFx` are the two shipped users; `ck.effigy.swing` predates the helper and still
hand-writes its ladder.

**Persistent — a store row.** State goes through `NetBus.RegisterStore(name, StoreOptions)` (the
`petfx` / `effigy` / `proxy` stores above): the store owns the announce latch, periodic refresh,
late-join replay and room/peer teardown; you keep a ledger of what a row *means* and apply it from
`OnSet` / `OnCleared`. The worked example is **`CompanionAura`** riding the `petfx` store: the
consumer registers one `AuraRecipe` at boot (`CompanionAura.Register`) and then pushes the desired
state every tick (`CompanionAura.SetActive(ownerUid, auraKey, active)`), which wraps
`CompanionPetFx.SyncOwnedFx` — the store latches the change, a guest's pet reaches the master as a
`ck.proxy.petfx` report and is re-announced under its owner, and a late joiner gets the row from
the scene-ready flush. Beastwhispering's `PetDotAuraDriver` (its `DotAuras` table: one aura row per
damage-over-time status) is that path end to end. A persistent row never needs a transient verb
beside it, and a transient never needs a store; whichever you pick, apply through ONE seam that
every path (owner-local, master, wire) lands in — a separate "local" code path is where the
LanternShare guest-skip class of bug lived.

**Guest net mirror.** `CompanionCombat` does not know the `ck.proxy.*` protocol:
its target/stance mirroring and hit replay go through an injected `ICompanionNetMirror`. The
consumer that speaks the proxy protocol sets the bond's factory once —
`companion.NetMirrorFactory = ProxyPets.NewGuestMirror;` (Beastwhispering's `Pet` ctor) — and
`AdoptBody` wires a fresh mirror per body. A consumer that never sets it (Hireling) sends **no**
proxy bytes at all, which is what closes the dual-install hijack: previously every consumer's
companion body carried the mirrors and a second consumer's follower could drive the pet protocol.
Relatedly, the bond's `CombatMandate` seam (`Companion.CombatMandate`) injects "is this fight
mandated?" into the anchor's convergent re-calm — null means the local rule (mandate ⇔ engaged
stance); `ProxyPets` wires its guest-instructed-lock validation there.

**Effigy extensibility.** Two consumer seams on the cosmetic-body layer:

- `CompanionEffigy.BodySettingsFactory` — set it to a `species => new MyEffigySettings(species)`
  factory to adjust effigy **body construction**: `EffigyBodySettings` is public
  (`CompanionSettingsDefaults` subclass; defaults pin the beast yaw at 180° and tag logs
  `[PUPPET/EFFIGY:<species>]`), and every `ICompanionSettings` member is overridable. A throwing or
  null-returning factory falls back to the kit default with a warn-once — never a bodiless effigy.
- The `ck.effigy.set` payload carries an **opaque third-field extension**
  (`NetProtocol.BuildEffigySet(species, tier, extension)` / the 4-out `ParseEffigySet`; it lands on
  `EffigyRow.Extension`). Empty appends nothing — the wire stays byte-identical — and every shipped
  parser ignores fields past the tier, so a future consumer field cannot break an old peer. The
  extension is the raw remainder (it may carry tabs); framing inside it is the consumer's.

`CompanionKit.Core` is a game-reference-free, unit-tested compute layer: weld policy, target selection,
the **follow policy** (`FollowPolicy` — the whole `CompanionBody` drive ladder: zone re-place vs
direct-drive vs normal following, plant/leash/grace rules, every threshold a named `FollowTuning`
field), the attribute codec (`CreatureAttributes.Flatten` / `Parse` is one save-safe string), the donor
table, and the expedition manifest and warm policy.

**Mount seams.** Two `ICompanionSettings` knobs exist for a consumer whose owner *rides*
the body (a mount mod) instead of being followed by it. Both default to today's behavior, so an
ordinary companion consumer never touches them:

| Knob | Default | What `false`/`true` buys a mount |
|---|---|---|
| `AnchorEnabled` | `true` | `false` = `Companion.Tick` skips anchor upkeep entirely — no ghost anchor is spawned, welded, or leashed. The bond runs anchor-less; the consumer owns combat embodiment (or has none). |
| `SuppressLeashWarp` | `false` | `true` = the follow policy never re-places or warps the body on **owner distance** (the >50 m zone trigger, the leash `TryWarp`, and the latched backstop) — a ridden body positions itself. The `!isOnNavMesh` re-place is kept: that half re-binds the agent to *a* navmesh and has nothing to do with owner distance. |

Related: `CompanionBody.LeashDistance` is genuinely honored at the zone trigger — the re-place fires at
`max(50, LeashDistance)`, so raising the leash past 50 m raises the zone trigger with it (the shipped
default of 14 resolves to exactly the historical 50).

**Honest strikes (`StrikeJudge`).** A companion's special attack, a scripted execute or
any other *synthesized* hit bypasses the physics pipeline, so without a judge it always connects —
blocks and dodge rolls would be meaningless against it. `CompanionKit.StrikeJudge` samples the target
at the moment the hit would land and rules on it, mirroring the engine's own connect rules:

```csharp
StrikeOutcome outcome = StrikeJudge.Judge(attackerPos, attacker.forward, target,
                                          maxRange: 4.5f, coneDegrees: 100f);
if (outcome == StrikeOutcome.Blocked)
    StrikeJudge.PlayBlock(target, source: myWeaponOrBody, damage, dir,
                          angleFromTargetForward, dealer, log: Log.LogMessage);
```

- Outcomes are `Landed` / `Blocked` / `Dodged` / `OutOfRange` / `OutOfAngle` / `Missed` — the check
  order picks the most useful reason, not engine chronology.
- A block needs the target facing you: 60° from its forward, 80° with a shield, strict less-than —
  the vanilla numbers. Dodge i-frames are physical (a rolling character's hitboxes are switched off),
  which `HitboxesAllInactive` reads directly; bodies with no hitbox rig fall back to the dodge flag.
- `maxRange <= 0` skips the range check (a projectile impact is already physical); `coneDegrees` of
  0 or 360 skips the facing check.
- `PlayBlock` expresses a judged block through the engine's own verb, so it produces the real thunk,
  stagger and FX while dealing nothing. **`source` must be a live MonoBehaviour** — the engine
  dereferences it for the FX position — and a blocking creature with no weapon of its own can throw
  inside the vanilla body, which is caught and reported to your `log` sink rather than escaping.
- The rules themselves are pure and unit-tested in `CompanionKit.Core.StrikeJudge`, so a consumer can
  judge sampled facts offline.

### Persistence — `CompanionPersistence<TState>` + `CompanionSaveStore<TState>` + `BodyAcquisition`

The companion save/re-form spine. The split rule: **the kit owns the LIFECYCLE** — the
sceneLoaded→player-ready state machine, the session-end teardown, the per-character file plumbing,
the ONE cold-load identity funnel, and the body-acquisition ladder with its retry budget — and
**the consumer owns the STATE**: `TState` is your schema, the codec is your hook, the kit never
sees a field of it.

```csharp
// 1. The store: per-character files under BepInEx/config/<prefix><uid>.txt
//    (BW: bw_pets_<uid>.txt via its SaveCodec; Hireling: hl_folk_<uid>.txt via FollowerCodec).
//    Crash-safe atomic writes (.tmp + swap), stale-.tmp cleanup on load.
var store = new CompanionSaveStore<MySave>(_host, "mymod_",
    encode: s => MyCodec.Serialize(s),                    // null state must yield your empty form
    decode: (content, path) => MyCodec.Deserialize(content, warn));

// 2. The ladder: keeps a bodiless companion hunting for a body. Rung ORDER is fixed
//    (expedition hold → your nearby rung → template cache → donor harvest → ghost stand-in once);
//    each rung is opt-in, and the donor-harvest retry budget (60s cooldown, 3 retries/scene
//    episode, 9 additive cycles/session — the LightProbes crash ceiling) is Core.HarvestPacing.
var ladder = new BodyAcquisition(new BodyAcquisitionSpec
{
    Host = _host, Runner = this, Noun = "follower",
    Live = () => _mine?.Companion, Player = () => LocalPlayer,
    SpeciesId = () => _mine?.Name, OnBodyBuilt = (body, isGhost) => Attach(body),
    TryBuildNearby = p => FindAndClone(p),                // your rung 1; null = miss, falls through
    UseTemplateCache = false, UseDonorHarvest = false, UseGhostStandIn = false,
});

// 3. The spine: forward SceneManager.sceneLoaded to it; hooks run in a fixed pipeline order
//    (per-scene resets → [gameplay] player-ready wait → owner-mismatch guard → OnBeforeLoad →
//    cold-load funnel OR OnInSessionArrival → OnAfterLoad → ladder kick; [main menu] OnSessionEnd
//    → teardown with save files untouched).
_persistence = new CompanionPersistence<MySave>(new CompanionPersistenceSpec<MySave>
{
    Host = _host, Runner = this, Noun = "follower", Store = store,
    LocalPlayer = () => LocalPlayer, HasLive = () => _mine != null,
    LiveBody = () => _mine?.Body, LiveOwnerUid = () => _mine?.OwnerUid,
    LiveSpeciesId = () => _mine?.Name, TeardownLive = p => TeardownLive(),
    SavedIdentityError = (saved, uid) => Validate(saved),   // non-null = REFUSED loudly, file kept
    AdoptSaved = (saved, player) => AdoptBodiless(saved, player),
    Acquisition = ladder,
});
void OnSceneLoaded(Scene s, LoadSceneMode m) => _persistence.OnSceneLoaded(s, m);
```

What the kit guarantees (all pure-tested in `CompanionKit.Core.ReformFlow`/`HarvestPacing`):

- **Scene routing** — additive loads (donor harvests) are ignored entirely; the transition/
  LowMemory shells between zones never tear the live companion down; ONLY the main menu is
  session-end (DDOL bodies must not leak into the next character's session).
- **The owner guard** — a live companion whose `OwnerUid` differs from the arriving player is torn
  down (save untouched) before that player's own file loads.
- **The cold-load identity gate** — every cold load funnels through your `SavedIdentityError`
  validator; a refusal logs loudly under `[PERSIST]` and keeps the file (the data is the player's
  bond — deleting is never the gate's call). Beastwhispering's stand-in-species gate rides this seam.
- **Ladder pacing** — the donor-harvest rung re-arms on a 60 s cooldown and immediately on a scene
  change, fires for ghost stand-in bodies too (upgrading them is the point), and PARKS loudly when
  the budget is spent. Spectral beats a crash.

**Save triggers stay yours** — the kit never decides *what counts as a change* (BW's heartbeat +
edge policy is `Beastwhispering.Core.SavePolicy` in its own tick; Hireling saves at recruit, zone
arrival and quit-to-menu). Log lines emit through YOUR host's logger, so a consumer's grep
contracts (BW's `[PERSIST]`/`[SAVE]` lines) survive the move byte-for-byte.

Consumers today: **Beastwhispering** (all four rungs, `bw_pets_<uid>.txt`, behavior-identical to
its pre-kit spine) and **Hireling** (recruit survives relaunch — nearby rung only,
`hl_folk_<uid>.txt`, position re-placed when reloading in the scene it was saved in).

### A native menu tab — `MenuTabInjector` + `MenuPanelBase`

Your companion needs a sheet somewhere. `MenuTabInjector` adds a real tab to the vanilla character
menu window (beside Inventory/Skills/…), with native LB/RB cycling, Esc-close and toggle visuals —
no custom UI framework, no window of your own. Register a spec, subclass a panel, and add two
Harmony patches; the tab machinery is the kit's, the content is entirely yours.

```csharp
private static readonly MenuTabSpec Tab = new MenuTabSpec   // ONE static instance — see below
{
    Label = "Companion",          // the tab button AND the window's section header
    PanelType = typeof(MyPanel),  // must derive from CompanionKit.MenuPanelBase
    Log = Plugin.Log,
    Tag = "[MYTAB]",              // your log-line prefix
};

[HarmonyPatch(typeof(CharacterUI), "Awake")]
internal static class UiAwakePatch
{
    [HarmonyPostfix] private static void Postfix(CharacterUI __instance)
        => MenuTabInjector.Inject(__instance, Tab);
}

[HarmonyPatch(typeof(CharacterUI), nameof(CharacterUI.ShowMenu),
    new[] { typeof(CharacterUI.MenuScreens), typeof(Item) })]
internal static class UiShowMenuPatch
{
    [HarmonyPostfix] private static void Postfix(CharacterUI __instance, CharacterUI.MenuScreens _menu)
        => MenuTabInjector.OnShowMenu(__instance, _menu, Tab);
}

internal class MyPanel : MenuPanelBase   // build your uGUI in an EnsureBuilt() from Show()
{
    public override void Show() { /* … */ base.Show(); }
}
```

`MenuTabInjector.ShowFor(player, Tab)` opens it directly (the headless dev-verb driver);
`TryGetScreen` hands back the synthetic screen id.

Three things are load-bearing and were learned the hard way:

- **The spec must be one long-lived static object.** The injector keys its per-`CharacterUI`
  bookkeeping on the spec's identity, so a freshly-built spec per call would re-inject on every
  `Awake`.
- **The patches stay yours, not the kit's.** Injection is opt-in per mod, and your kill-switch keeps
  working.
- **Your panel must derive from `MenuPanelBase`.** Its `StartInit` walks the parent chain by hand to
  find the `MenuPanelHolder` and `RegisterChildMenu`s itself. A from-scratch panel born INACTIVE can
  never find its holder with `GetComponentInParent` — `UIElement.Show()` runs `Start()` *before*
  activating the GameObject, and Unity 2020's `GetComponentInParent` skips inactive objects — and a
  panel missing from the holder's census makes the holder close the **whole window** the instant the
  previous tab finishes fading out.

The tab index is taken from the arrays' CURRENT length rather than a hardcoded slot, so this
composes with any other mod that grows the same arrays (Transmorphic does, via reflection).

### Player stat buffs — `PlayerStatBuff`

The "lay `StatStack`s on the owner, re-derive them every tick, survive the player Character being
replaced" applier. Stat stacks are not game-saved and die with the Character on a zone change or
reload, so the only correct shape is convergent: `Sync(player, desired)` no-ops while the same
player already holds exactly `desired`, and re-lays everything on a fresh Character.

```csharp
private static readonly PlayerStatBuff _buff = new PlayerStatBuff();

// each tick — the desired set is recomputed from live state, not remembered
var want = new List<StatStackSpec> {
    new StatStackSpec(stats.m_damageTypesModifier[0], "MyMod.Bond", 0.04f, multiplier: true, "Physical"),
};
StatBuffResult r = _buff.Sync(player, want, onSkip: s => Log.LogWarning($"unresolved {s.Label}"));
if (r.Change != StatBuffChange.NoOp) Log.LogMessage($"applied {r.AppliedCount}"
    + (r.SamePlayer ? "" : " (fresh player)"));
```

`Clear()` removes only *our* stacks (a stat someone else also writes keeps theirs), and is safe on a
destroyed Character. `PlayerStatBuff.OurStack(stat, sourceId, mult)` is the `[ours +0.04]` readback
for a dump verb — worth wiring, because it separates "we decided wrong" from "we wrote it wrong".
Pass `retainOwnerWhenEmpty: false` if you want ownership claimed only when stacks are actually laid.

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

### Name drift — reading older docs

CompanionKit was lifted out of Beastwhispering on 2026-07-06 (`docs/companionkit-plan.md`), and
pre-extraction docs, test reports and commit messages use the old names. The map:

| Today | Was (Beastwhispering) |
|---|---|
| `BodyFactory` | `CreatureCloner` |
| `CompanionBody` | `PuppetFollower` |
| `CompanionAnchor` (now an INSTANCE per bond) | `PetAnchor` (static) |
| `CompanionCombat` (stance lives on the `Companion`/Pet aggregate) | `PetCombat` |
| `AttributeCapture` | `StatCapture` |
| `CreatureAttributes` (Core) | `PetAttributes` (`PetStatTuning` stayed in Beastwhispering) |
| `TargetPick` (Core) | `Core.PetCommand` |
| `DonorHarvest` + guards | same names, moved as-is (and on to DonorKit on 2026-07-26) |

`LocoRig.cs` (added 2026-08-18) has no old name: it centralizes "who actually renders this body" —
it finds the skin-driving animator (`CompanionBody.FindSkinAnimator`) and writes locomotion params
to it AND to the owned root animator when they differ. It is the one seam a future rig-class fix
should touch instead of re-scattering parameter writes.

## Compatibility

Outward must be on its **Mono** Steam beta branch, not the default IL2CPP build. If the game runs but no
BepInEx mods load and there's no crash log, this is almost always why (Steam → game Properties → Betas →
select `mono`).

## See also

- [Kits index](./README.md) · [Wiki home](../README.md)
- Dependencies: [ForgeKit](./forgekit.md), [AggroKit](./aggrokit.md), [NetKit](./netkit.md)
- Built on top: [SpawnKit](./spawnkit.md)
- Mods that use it: [Beastwhispering](../mods/beastwhispering/README.md), [Hireling](../mods/hireling.md)
