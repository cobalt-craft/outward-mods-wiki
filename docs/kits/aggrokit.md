# AggroKit — aggro / threat control for Outward mods

**AggroKit** is a kit (reusable library) for Outward that gives mods precise control over what
enemies target — force an enemy onto a victim, calm it, taunt a group, or make two hostiles fight
each other — all built on the game's own AI APIs. It also ships a small, always-on fix for a
vanilla enemy-squad targeting glitch. It has no player-facing UI; you install it because a mod you
want depends on it.

**At a glance**
- Type: reusable library (kit)
- Requires: BepInEx 5 (Mono branch), [ForgeKit](forgekit.md)
- Config: `BepInEx/config/cobalt.aggrokit.cfg`
- Commands: `BepInEx/config/ak_cmd.txt`

## For players

You don't interact with AggroKit directly — it arrives as a dependency of a mod (for example
[Beastwhispering](../mods/beastwhispering/README.md), whose companion uses it to hold enemy
attention). The one thing it changes on its own is a small **bug fix**: it stops a member of an
enemy squad from occasionally getting stuck "fighting itself" (standing in a combat pose, attacking
nothing) when combat spreads through the group. That fix is on by default and can be turned off in
the config if you want the raw vanilla behavior back.

## How it works

AggroKit is two things:

**1. A default-on engine fix.** When one enemy in a squad enters combat, the game can spread that
combat to nearby squad-mates ("contagion"). In a corner case this hands a squad-mate *its own*
identity as a target, so it stands in a fake fight — attacking nothing — for up to about a minute,
until a real attacker's blow snaps it out of it. AggroKit blocks that self-target assignment. It is
unreachable in normal play (a squad-mate is never a valid target), so leaving the fix on is safe.

**2. Patch-free aggro tools.** A set of operations built entirely on public engine calls — no
combat patching — that a mod can invoke to move threat around:

- **Force target** — make one enemy lock onto a specific character right now, entering its combat
  state if needed. The lock is sticky (vanilla AI only re-acquires when it has no lock).
- **Calm** — drop an enemy's target and send it back to wandering.
- **Taunt** — redirect every hostile in a radius that's currently fighting one character onto
  another.
- **Feud** — make any two AIs fight each other, installing a targetability override for the pair
  when the faction rules would normally forbid it.
- **Diagnostics** — dump every AI near a point (name, faction, state, locked target, distance) and
  dump an individual AI's state machine.

On top of those, a single Harmony patch on the engine's `IsTargetable(Character)` gate lets a mod
mark any attacker→target pair (or any target globally) as attackable or unattackable without
touching factions, plus a set of **detection-side controls** — veto detection of a character (true
stealth), redirect detections onto a decoy, or make a character's hits generate no aggro.

### Known limitations

- **Ranged and elemental damage bypass the targetability gate.** Projectiles and blasts read a
  faction snapshot taken when they're fired, not the live `IsTargetable` check — so a character made
  "untargetable" still takes full ranged damage even though melee passes through.
- **The plain untargetable flag flaps in combat.** The initial detection scan looks at factions
  only, so an "untargetable" character is still *noticed*; mid-fight this cycles the enemy in and out
  of combat (~6 s) rather than ending it cleanly. Layer the detection veto on top for true peace.
- **Squad target hand-outs can bypass stealth.** Squad tactics can assign a target with no detection
  scoring at all. AggroKit's detection veto and redirect controls do cover this path, but the plain
  detectability toggle does not.

## Settings

`BepInEx/config/cobalt.aggrokit.cfg`, generated on first launch.

| Section | Key | Default | Effect |
|---|---|---|---|
| `Dev` | `EnableCommandFile` | `true` | Poll `ak_cmd.txt` for dev commands. |
| `Dev` | `EnableDumpKey` | `true` | Whether the dump key logs an aggro dump of nearby AI. |
| `Dev` | `DumpKey` | `F3` | Key that logs an aggro dump of AI near the player (rebindable; needs `EnableDumpKey`). |
| `Research` | `EnableObservation` | `false` | Re-arm the purely-observational Harmony taps that record the `[AGGRO]` event buffer (target changes, state switches, squad hand-outs, faction flips). Off by default so these hot AI-tick paths stay unpatched. Applied at load — changing it needs a game restart. |
| `Fixes` | `BlockSquadSelfTarget` | `true` | Kill-switch for the always-on squad self-target fix. On is safe; turn off only to observe the raw engine bug. |

The detection-side control patches and the targetability override gate are always installed but stay
inert until a command or a mod activates them, so they cost nothing when unused.

### Example configuration

`BepInEx/config/cobalt.aggrokit.cfg` — created on first launch. Excerpt:

```ini
[Dev]
EnableCommandFile = true
DumpKey = F3

[Research]
EnableObservation = false

[Fixes]
BlockSquadSelfTarget = true
```

A full generated example lives at `tests/fixtures/config/cobalt.aggrokit.cfg`, and the shared-settings
overlay is `config/shared/cobalt.aggrokit.cfg.overlay` (see `config/README.md`).

## Commands

Write a verb into `BepInEx/config/ak_cmd.txt` and it runs on the next poll (even while the game is
paused). Unknown verb or `help` lists them all. The dump key (default **F3**) runs `aggrodump`.

AggroKit uses the command channel **only** — it deliberately does not adopt ForgeKit's shared
`CommonVerbs` pack, so its verbs are exactly the aggro-focused set below.

| Verb | Does |
|---|---|
| `aggrodump [radius]` | Dump every AI near the player (name, faction, state, target, squad, distance) + the player's targetability flags. |
| `aistates [name\|nearest]` | Dump one AI's state machine. |
| `aggrolog [n\|on\|off\|clear]` | Show the last `n` recorded `[AGGRO]` events, toggle live log mirroring, or clear the buffer. |
| `watch <name\|nearest\|off>` | Watch one AI's (state, target) changes and log each one with how long the previous held. |
| `forcetarget [name\|nearest]` | Force an AI onto the player and auto-watch it. |
| `feud <nameA> <nameB> [nooverride\|both]` | Make two AIs fight; installs a targetability override for the pair unless `nooverride`; `both` forces both directions. |
| `taunt [radius]` | Redirect every AI in radius onto the player. |
| `calm [radius]` | Drop the target of every AI in radius. |
| `untargetable` / `targetable` | Toggle the player's `QuestNonTargetable` flag. |
| `undetectable` / `detectable` | Toggle the player's detectability to 0 / 1. |
| `shieldme` / `unshieldme` | Override so nobody may target the player / clear it. |
| `clearoverrides` | Clear all targetability overrides. |
| `stealthme [off]` | Veto AI detection of the player (true stealth). |
| `decoy <name>\|off` | Redirect detections of the player onto another AI. |
| `noaggro [off]` | The player's hits generate no aggro. |
| `status` | One-glance status: player flags, live overrides, controls, watches, event buffer. |
| `restore` | Reset everything — player targetable + detectable, all overrides/controls/watches cleared. |
| `selftest` | Zero-interaction environment checks (`[SELFTEST] PASS/FAIL … DONE`). |

Dev-staged overrides and controls (`feud`, `shieldme`, `decoy`, …) are cleared automatically on a
real zone or town transition, so nothing you stage in a test session leaks into normal play.
Only *dev-staged* state is swept — a control a mod installed under its own owner tag survives; see
[ownership and what survives a scene load](#ownership-and-what-survives-a-scene-load).

## For modders

Depend on AggroKit the usual way — a project reference plus a hard dependency so BepInEx loads it
first:

```csharp
[BepInPlugin(GUID, NAME, VERSION)]
[BepInDependency(AggroKit.Plugin.GUID)]   // "cobalt.aggrokit"
public class Plugin : BaseUnityPlugin { … }
```

### The canonical aggro operations

`AggroKit.AggroTools` is the single home of the force-target / calm / taunt primitives. Other kits
delegate here rather than reimplementing them — CompanionKit's combat anchor and Beastwhispering's
taunt controller both call these, so a change to the recipe lands in one place. All are static and
safe to call from any mod (they refuse rather than throw on half-initialized characters).

```csharp
using AggroKit;

// Make an enemy lock onto a victim right now (enters combat if needed).
// Returns true if the AI ended up in a combat state with the target locked.
bool locked = AggroTools.ForceTarget(enemyAi, victimCharacter);

// Drop its target and send it back to wandering.
AggroTools.Calm(enemyAi);

// Redirect everyone in radius who's fighting `from` onto `to` (from == null = anyone in combat).
// Returns how many AIs were redirected.
int n = AggroTools.Taunt(center, radius, to: myAnchor, from: player);

// Soft player toggles (engine-native flags).
AggroTools.SetUntargetable(player, true);
AggroTools.SetUndetectable(player, true);

// Enumerate live AI near a point, optionally excluding player-faction and one specific character.
foreach (CharacterAI ai in AggroTools.AisInRange(center, radius,
             excludePlayerFaction: true, exclude: myPetAnchor))
    …
```

`AggroTools.Taunt` is a one-shot radius sweep. For a taunt that *holds*, see below.

### Sustained taunts (`TauntController`)

A set lock can be stolen back — a hurt AI re-rolls its target, block-retargeting happens, and squads
hand out preferred targets — so pinning one enemy onto one character for a duration is a re-assert
loop, not a single call. `AggroKit.TauntController` is that loop: it re-asserts every 0.3 s, which
out-paces the AI's own 0.5 s target re-poll, so a steal is corrected before the AI acts on it.

```csharp
// Pin `enemy` onto `myAnchor` for 8 seconds; release it if the enemy gets further than 30 m away.
TauntController.Hold(protect: myAnchor, enemy: enemy, seconds: 8f,
                     leashMeters: 30f, owner: MyOwner, log: Log.LogMessage);

TauntController.ReleaseOwnedBy(MyOwner, "teardown");   // drop just yours
TauntController.ReleaseAll("restore");                 // the hammer: every owner
bool mine = TauntController.CountOwnedBy(MyOwner) > 0;
string forensics = TauntController.ForensicsFor(MyOwner);
```

- **One hold per protected character.** A second `Hold` for the same character replaces its hold;
  other characters' holds stand, so a host simulating several companions can't have one stomp another.
- **The leash is your policy, not a kit setting.** `leashMeters` is a parameter because the distance
  law belongs to whoever owns the body being defended; `<= 0` disables the rule entirely.
- **Releases:** duration expiry, the enemy dying or vanishing, the protected character going down,
  the leash, or an explicit release. A release only *stops asserting* — it never forces a calm, so the
  enemy simply fights whoever the vanilla rules pick next.
- **Ownership** follows the same rule as everything else below: `DevOwner` holds are swept at each
  non-additive scene load, yours survive, and `ReleaseAll` drops everything.
- AggroKit ticks the loop itself; a consumer may tick it too (`TauntController.Tick()` is clock-gated,
  so a second call in the same frame asserts nothing extra).

Accepted limitation: squad tactics can re-race the hold in pack fights — it is a sticky suggestion,
not an absolute.

### Targetability overrides

`AggroKit.TargetableOverrides` backs the one Harmony patch AggroKit ships — a postfix on
`TargetingSystem.IsTargetable(Character)`. Force a specific attacker→target pair (or any→target)
attackable or not, without changing factions:

```csharp
TargetableOverrides.Set(attacker, target, targetable: true, owner: MyOwner);   // this pair may fight
TargetableOverrides.SetForAll(target, targetable: false, owner: MyOwner);      // nobody may target `target`
TargetableOverrides.ClearOwnedBy(MyOwner);   // tear down just yours
TargetableOverrides.ClearAll();              // the big hammer: everything, every owner
```

A `true` override only makes a pair *attackable* — it does not make the detection scan find them.
Pair it with `AggroTools.ForceTarget` to start the fight. Mind the ranged-damage limitation above:
this gate governs lock-on and melee eligibility, not projectiles.

### Detection-side controls and observation

`AggroKit.AggroEvents` exposes the finer detection-pipeline hooks used by the `stealthme` / `decoy` /
`noaggro` commands — `SetDetectionVeto`, `SetDetectionRedirect`, `SetNoAggroDealer` — plus C# events
(`Detected`, `Hurt`, `TargetChanged`, `StateChanged`) a mod can subscribe to. These fall into two
groups. `Detected` and `Hurt` fire whenever an AI notices or is struck by something — their patches
are always installed, alongside the veto/redirect/no-aggro *controls*, which likewise run regardless.
`TargetChanged` and `StateChanged`, together with the `[AGGRO]` recording buffer, only fill when
observation is armed (`[Research] EnableObservation`, or `aggrolog on`), since those taps are the
ones that stay unpatched by default.

```csharp
AggroEvents.SetDetectionVeto(player, on: true, owner: MyOwner);
AggroEvents.SetDetectionRedirect(from: player, to: myAnchor, owner: MyOwner);
AggroEvents.SetNoAggroDealer(player, on: true, owner: MyOwner);
AggroEvents.ClearControlsOwnedBy(MyOwner);
```

### Ownership and what survives a scene load

Every control, every targetability override and every taunt hold carries an **owner tag**. AggroKit sweeps its own
dev staging on each non-additive scene load (a zone change, a town door, a fast travel) — and only
its own:

- `owner` defaults to `AggroEvents.DevOwner` / `TargetableOverrides.DevOwner` (`"ak_cmd"`), the tag
  the `ak_cmd.txt` verbs use. **DevOwner-tagged state is cleared at every scene load.**
- Anything tagged with your own string is **retained across scene loads** and is yours to remove —
  `ClearControlsOwnedBy(tag)` / `ClearOwnedBy(tag)`, or the un-set overload of the same call.
- **The per-entry clears are ownership-BLIND.** `Clear(attacker, target)`, `ClearForAll(target)`,
  `ClearDetectionRedirect(from)` and the `on: false` form of the `Set*` calls remove whatever is
  registered for that character, whoever owns it — so a dev verb (`unshieldme`, `decoy off`,
  `stealthme off`) can drop a consumer-owned entry for the same character. Ownership decides what
  the *scene-load sweep* keeps, not who may remove a specific entry.
- `ClearAllControls()` and `TargetableOverrides.ClearAll()` ignore ownership and drop everything;
  they back the `restore` / `clearoverrides` verbs. Don't call them from a consumer unless you mean
  to wipe other mods' state too.

The sweep says what it did, so you can see your own state survive:

```
[AggroKit] scene 'CierzoNewTerrain' loaded (Single) -- cleared 2 dev-staged
(overrides=1 controls=1 of (detectVeto=1 redirect=1 noAggroDealers=0)), retained 1 consumer-owned.
```

Pick a stable tag — your plugin GUID is the obvious choice:

```csharp
private const string MyOwner = "cobalt.mymod";
```

Two caveats that predate ownership and still hold: overrides and controls are keyed by
`Character.UID`, which does not survive a save reload for every character class, and none of this
state is persisted — re-install your controls when your own state re-initialises.

## See also

- [Kits index](./README.md)
- [ForgeKit](forgekit.md) — the dev tooling AggroKit's command channel is built on
- [CompanionKit](companionkit.md) — its combat anchor delegates to AggroKit's operations
- [Beastwhispering](../mods/beastwhispering/README.md) — the mod that leans on it most
- [Wiki home](../README.md)
