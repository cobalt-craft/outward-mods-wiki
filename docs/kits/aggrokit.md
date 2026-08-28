# AggroKit — aggro / threat control for Outward mods

**AggroKit** is a kit (reusable library) for Outward that gives mods precise control over what
enemies target — force an enemy onto a victim, calm it, taunt a group, or make two hostiles fight
each other — all built on the game's own AI APIs. It has no player-facing UI; you install it because
a mod you want depends on it.

Out of the box AggroKit changes **nothing** about how your game behaves: every patch it installs is
inert until a mod or a command activates it.

**At a glance**
- Type: reusable library (kit)
- Requires: BepInEx 5 (Mono branch), [ForgeKit](forgekit.md)
- Config: `BepInEx/config/cobalt.aggrokit.cfg`
- Commands: `BepInEx/config/ak_cmd.txt`

## For players

You don't interact with AggroKit directly — it arrives as a dependency of a mod (for example
[Beastwhispering](../mods/beastwhispering/README.md), whose companion uses it to hold enemy
attention). On its own it changes nothing about your game.

**If you are bisecting a suspected mod conflict**, set `EnablePatches = false` in
`BepInEx/config/cobalt.aggrokit.cfg` and relaunch. AggroKit then patches nothing at all, so you can
rule it in or out without uninstalling it (which would also disable every mod that depends on it).
Its boot log lines — `[AggroKit] patches:` and `[AggroKit] gates:` — say exactly which vanilla
methods it replaced and why, so a log is enough to answer the question.

## How it works

AggroKit is two things:

**1. An opt-in engine fix (`BlockSquadSelfTarget`, default OFF).** When one enemy in a squad enters
combat, the game can spread that combat to nearby squad-mates ("contagion"). In a corner case this
hands a squad-mate *its own* identity as a target, so it stands in a fake fight — attacking nothing —
for up to about a minute, until a real attacker's blow snaps it out of it. AggroKit can block that
self-target assignment.

It ships **off**, and the reason is worth stating because it cuts the other way from how it reads:
the corner case is unreachable in unmodified play, and the block is implemented as a Harmony prefix
that skips the original method — which in BepInEx also suppresses *any other mod's* prefix on the
same method. So the shipped default was paying a cross-mod cost for a fix whose trigger cannot be
demonstrated. Turn it on if you actually see an enemy stuck in a fake fight; it takes effect without
a restart.

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
| `Fixes` | `EnablePatches` | `true` | Master switch for **every** Harmony patch AggroKit installs. Off = AggroKit patches nothing at all, while the library calls and `ak_cmd.txt` verbs keep working — set it off to rule AggroKit out of a mod-conflict bisection without uninstalling it. Applied at load — changing it needs a game restart. |
| `Fixes` | `BlockSquadSelfTarget` | `false` | Opt-in squad self-target fix (see *How it works*). Off by default: the case is unreachable in unmodified play, and the block suppresses other mods' patches on the same method. Read live — no restart needed. |

With `EnablePatches` on (the default), the detection-side control patches and the targetability
override gate are installed but stay inert until a command or a mod activates them, so they cost
nothing when unused.

Edit the file with the game closed: AggroKit does not take ForgeKit's shared verb pack, so it has no
`reloadcfg` verb and every change here — including a `DumpKey` rebind — is picked up at the next
launch.

### Example configuration

`BepInEx/config/cobalt.aggrokit.cfg` — created on first launch. Excerpt:

```ini
[Dev]
EnableCommandFile = true
EnableDumpKey = true
DumpKey = F3

[Research]
EnableObservation = false

[Fixes]
EnablePatches = true
BlockSquadSelfTarget = false
```

A full generated example lives at `tests/fixtures/config/cobalt.aggrokit.cfg`, and the shared-settings
overlay is `config/shared/cobalt.aggrokit.cfg.overlay` (see `config/README.md`).

## Commands

Write a verb into `BepInEx/config/ak_cmd.txt` and it runs on the next poll (even while the game is
paused). Unknown verb or `help` lists them all. The dump key (default **F3**) runs `aggrodump`.

AggroKit uses the command channel **only** — it deliberately does not adopt ForgeKit's shared
`CommonVerbs` pack, so its own verbs are exactly the aggro-focused set below (plus the
`script` / `scriptcancel` / `scriptstatus` sequencing verbs every ForgeKit channel carries).

| Verb | Does |
|---|---|
| `aggrodump [radius]` | Dump every AI near the player (name, faction, state, target, squad, distance) + the player's targetability flags. |
| `aistates [name\|nearest]` | Dump one AI's state machine. |
| `aggrolog [n\|on\|off\|clear]` | Show the last `n` recorded `[AGGRO]` events, toggle live log mirroring, or clear the buffer. |
| `watch <name\|nearest\|off>` | Watch one AI's (state, target) changes and log each one with how long the previous held. |
| `forcetarget [name\|nearest]` | Force an AI onto the player and auto-watch it. Reports the outcome and, on a failure, the reason (see `attack`). |
| `attack <victim> [attacker] [holdSeconds]` (alias `aggroonto`) | **Make something attack a chosen victim, and say why if it doesn't.** `victim` is a name substring, `uid:xxxx`, `me` (the player), or `pet`/`ally` (the nearest player-faction AI that is not you — a companion anchor, a hireling, a summon). `attacker` defaults to the nearest non-player-faction AI within 40 m. A pair targetability override is installed *only* when the faction matrix would veto the lock, the attacker's squad is alerted the way a vanilla detection does, and the pin is **sustained** for `holdSeconds` (default 20, `0` = one-shot) so AISCombat's 0.5 s re-poll and the 50 % on-hurt target switch cannot steal it straight back. Every run logs `N retargeted` plus a named outcome — `Locked`, `NoAttacker`, `NoVictim`, `AttackerHasNoTargeting`, `VictimNotAlive`, `VictimHasNoLockingPoint`, `BlockedByOverride`, `NoCombatState`, `LockDidNotHold` — at **Warning** level when nothing was retargeted. |
| `feud <nameA> <nameB> [nooverride\|both]` | Make two AIs fight; installs a targetability override for the pair unless `nooverride`; `both` forces both directions. |
| `taunt [radius]` | Redirect every AI in radius onto the player. |
| `calm [radius]` | Drop the target of every AI in radius. |
| `truce [seconds] [name\|nearest]` | One AI tolerates the player for N seconds (default 60): no scan/squad acquisition, calmed once if already on you; **your hit on it ends the truce**. Other AIs unaffected. |
| `truceoff [name\|nearest\|all]` | End one truce, or all of them. |
| `truces` | List live truces (beast→player, owner, seconds left). |
| `untargetable` / `targetable` | Toggle the player's `QuestNonTargetable` flag. |
| `undetectable` / `detectable` | Toggle the player's detectability to 0 / 1. |
| `shieldme` / `unshieldme` | Override so nobody may target the player / clear it. |
| `clearoverrides` | Clear all targetability overrides. |
| `protect <name\|nearest>` / `unprotect` | Make an AI un-attackable by everyone — AI acquisition, melee, lock-on, skill sub-effects — and release the locks already on it / undo. |
| `stealthme [off]` | Veto AI detection of the player (true stealth). |
| `decoy <name>\|off` | Redirect detections of the player onto another AI. |
| `noaggro [off]` | The player's hits generate no aggro. |
| `status` | One-glance status: player flags, live overrides, controls, watches, event buffer. |
| `restore` | Reset everything — player targetable + detectable, all overrides/controls/watches cleared. |
| `selftest` | Zero-interaction environment checks (`[SELFTEST] PASS/FAIL … DONE`). |

**Staging a fight against a companion.** The reliable form is `attack` on AggroKit's own channel —
one line, sustained, and self-evidencing:

```
attack pet            # nearest hostile fights your companion for 20 s
attack pet Hyena 30   # name the attacker, hold it for 30 s
attack me             # the classic: bring the nearest hostile onto you
```

A success reads `[AggroKit] attack: 1 retargeted — 'Hyena#3c36' [Hounds] -> 'Summoned Ghost#NxPw'
[Player] dist=2.4m state=AISCombatMeleeHound hold=20s [Locked] — locked …`; a miss reads
`0 retargeted` with the reason and its remedy, as a **warning**. Confirm with `aggrodump` — the
attacker's `target=` should be the companion, not you.

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

### Truces (`AggroTruce`)

The opposite of a taunt: one AI **tolerates one character** for a while. Built for Beastwhispering's
failed-tame outcome ("the beast refuses you — but lets you be, for now"), pet-agnostic.

```csharp
// `beast` ignores `player` for 60 s; a hit from `player` OR from `myAnchor` ends it.
AggroTruce.Hold(beast, player, seconds: 60f, owner: MyOwner, log: Log.LogMessage, myAnchor);

AggroTruce.Release(beast, player, "fail-aggro");   // end one pair (no Calm, no ForceTarget)
AggroTruce.ReleaseOwnedBy(MyOwner, "teardown");    // drop just yours
AggroTruce.ReleaseAll("restore");                  // the hammer
bool held = AggroTruce.IsHeld(beast, player);
string forensics = AggroTruce.Forensics;
```

- **Pair-keyed**, unlike the global detection veto: the truced AI still fights everyone else, and
  every other AI still fights the truced character. The squad may hand that character to every
  *other* member — the pack does not honour one member's truce.
- **What it blocks:** scan acquisition (`AICEnemyDetection.Detected`) and squad hand-outs /
  contagion (`AISCombat.SetPreferredTarget`) — the same two prefixes the detection veto uses. At
  `Hold`, an AI already locked on the character is calmed once.
- **What breaks it:** a hit on the AI by the character or by any `alsoBreaks` character you name
  (a pet's anchor, typically — the kit cannot know what a "pet" is). The break happens in the
  `CharHurt` / `Defense` prefixes *before* the vanilla response runs, so the hit that ends the truce
  is also the hit that aggroes the AI — exactly as if no truce had existed. A stranger's hit does
  not break it.
- **Releases:** expiry, the beast or character ceasing to exist (two consecutive 0.5 s sweeps
  without the uid resolving), or an explicit release.
- **Ownership** as everywhere else: `DevOwner` truces (the `truce` verb) are swept at a scene
  load; yours survive a zone door and are yours to release. `restore` drops everything.
- **Not covered:** Suspicious/investigate behaviour — the AI may still walk over and look; it
  never locks.

Bookkeeping (pair key, expiry, break rules, owner sweep) lives in the pure `TruceTable`, which
`tests/AggroKit.Tests` covers without a game boot.

### Targetability overrides

`AggroKit.TargetableOverrides` backs the targetability gate — a postfix on
`TargetingSystem.IsTargetable(Character)`. Force a specific attacker→target pair (or any→target)
attackable or not, without changing factions:

```csharp
TargetableOverrides.Set(attacker, target, targetable: true, owner: MyOwner);   // this pair may fight
TargetableOverrides.SetForAll(target, targetable: false, owner: MyOwner);      // nobody may target `target`
TargetableOverrides.ClearOwnedBy(MyOwner);   // tear down just yours
TargetableOverrides.ClearAll();              // the big hammer: everything, every owner
```

A `true` override only makes a pair *attackable* — it does not make the detection scan find them.
Pair it with `AggroTools.ForceTarget` to start the fight.

**Resolution order** is pair > for-all > vanilla: a `SetForAll(target, false)` with a
`Set(bandit, target, true)` on top means *only* that bandit may attack `target`.

**What a `false` override stops** (2026-08-22): lock-on acquisition, melee weapon hits, physic
projectiles, `CheckIfCombatWorthy`, AI combat-state retention (the `IsTargetable(Character)` postfix),
**AI acquisition** — `AICEnemyDetection.Detected` / `CharHurt` and the squad's `SetPreferredTarget`
refuse a blocked target, so a protected character is never locked in the first place instead of
flapping in and out of combat — skill/status sub-effects (`SubEffect.IsTargetable`), and
`AggroTools.ForceTarget`. Still open: `RaycastProjectile` volleys with `HitEnemiesOnly`, which are
faction-keyed.

#### Protecting a character

The convenience for the common case — "nobody may attack this NPC" (a quest giver, a road
merchant the player's pet would otherwise bite):

```csharp
const string MyOwner = "mymod";
AggroTools.Protect(merchant, MyOwner);                        // for-all false + release every AI lock on him
TargetableOverrides.Set(bandit, merchant, true, MyOwner);     // the ambushers may still attack (pair wins)
bool blocked = AggroTools.IsProtected(petAnchor, merchant);   // the read CompanionKit's anchor/pet picker use
AggroTools.Unprotect(merchant, MyOwner);                      // owner-checked: another owner's shield stays
TargetableOverrides.ClearOwnedBy(MyOwner);                    // and your pair re-allows
```

Consumer-owned protection **survives scene loads** (only the dev-verb owner is swept) — call
`Unprotect` when the character despawns. `AggroTools.ReleaseLocksOn(target)` is the lock sweep on
its own. CompanionKit honours the gate at its two choke points (the anchor's upkeep releases a held
lock on a protected character and never unifies onto one; the pet's target ladder refuses one as a
commanded / anchor-defend / player-engaged candidate), so a Beastwhispering pet will not fight a
protected NPC.

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
