# ForgeKit — shared dev-tooling for Outward mods

**ForgeKit** is a kit (reusable library) for Outward that gives mod authors a common set of
development tools: a file-driven dev-command loop, a self-test harness, on-screen toasts, a
player-ready lifecycle wait, config-backed data tables, a cross-mod keybind registry, and a pack of
ready-made dev verbs. It is the bottom layer every other kit and mod in this family rests on. It has
no player-facing surface of its own.

**At a glance**
- Type: reusable library (kit)
- GUID: `cobalt.forgekit`
- Requires: BepInEx 5 (Outward's Mono branch)
- Dependencies: none (no SideLoader, no Harmony, no other kit)
- Config: none
- Commands: none of its own — it *provides* the command-channel machinery other mods use

## For players

ForgeKit itself does nothing on its own — you install it because another mod declares it as a
requirement. But every mod built on it exposes one thing you *can* touch directly, with nothing more
than a text editor: **the command channel.**

Each mod that uses ForgeKit watches its own plain-text file under `BepInEx/config/` — the names are
not uniform (`bw_cmd.txt` for Beastwhispering, `ck_cmd.txt` for CompanionKit, `SpawnKit_cmd.txt`,
`DangerousRoads_cmd.txt`, and so on; each mod's page names its own). While the game is running, open
that file in any text editor, type a verb
on its own line, save the file — and the mod runs it on its next poll (a fraction of a second later,
even if the game is paused). This is a **file pipe**: your text editor and the running game are two
separate programs, and the file is the mailbox between them. No console, no cheat menu, nothing
injected into the game — just a file getting written and read.

Type `help` and save to see the full list of verbs that mod's channel understands, with a short
description of each. Most mods only expose a handful of gameplay-relevant verbs this way (things
like teleporting, spawning an item, or adjusting the time of day) — this isn't a general cheat
console, it's whatever the mod's author chose to expose for testing and troubleshooting, and it
varies mod to mod. If you're playing co-op, note that some verbs are host-only (moving the whole
party's clock or location) precisely because a guest firing them would desync the session.

Because it is only a file, the same loop can be automated: anything that can write that file — a
script, a watcher, a helper on another machine — can drive a mod's verbs without the player typing
anything.

### Turning the logging up (or down)

Every mod built on ForgeKit gets one setting that controls **how much it writes to the log**, in its
own config file at `BepInEx/config/<guid>.cfg`:

```ini
[Diag]
## How much this mod writes to the log.
# Setting type: LogTier
# Default value: Verbose
# Acceptable values: Quiet, Normal, Verbose, Trace
LogLevel = Verbose
```

| Value | What you get |
|---|---|
| `Quiet` | Warnings and errors only. |
| `Normal` | Adds the mod's start-up lines, on-screen notices and self-test results. |
| `Verbose` | **Default.** Adds the normal per-feature diagnostics. |
| `Trace` | Adds per-tick detail. **This is what a bug report should be captured at.** |

**The default is deliberately chatty.** These mods are still being actively debugged, and a bug
report that arrives with the diagnostics already in it is worth more than a small log file. If your
logs are bigger than you'd like, set `LogLevel = Normal` — nothing about the mod changes, it just
says less.

It is per mod, so you can turn one mod all the way up without drowning in every other mod's output.
The setting is re-read while the game runs — save the file and it takes effect, no restart.

**One extra step for `Trace` only:** BepInEx's own log filter drops that level before it ever reaches
the file. Open `BepInEx/config/BepInEx.cfg` and make sure the `[Logging.Disk]` section lists `Debug`:

```ini
[Logging.Disk]
LogLevels = Fatal, Error, Warning, Message, Info, Debug
```

(`All` works too.) `Verbose` needs no such edit — stock BepInEx already passes `Info`. If you do
forget, the mod says so once in the log rather than leaving you guessing.

## What's in it

| Piece | What it gives a modder |
|---|---|
| `CommandChannel` + `CommandRegistry` | A file-driven dev-command loop: write a verb into `BepInEx/config/<yourmod>_cmd.txt` and it runs on the next poll. The poll runs on unscaled time, so it still fires while the game is paused. Unknown verb (or `help`) prints every registered verb with its help text. |
| `VerbHost` / `VerbContext` | A thin wrapper over the registry that adds a per-verb prologue: resolve the local player, an optional Photon master-only gate, and a per-verb error tag. |
| `SelfTestHarness` | The `[SELFTEST] BEGIN` / `PASS:` / `FAIL:` / `SKIP:` / `DONE pass= fail= skip=` report shape — wire up a handful of `Check(name, condition)` calls and get one consistent, greppable self-test report. Use `Skip(name, why)` (or `CheckIf(canRun, …)`) for a check that cannot run *here* — a selftest at the main menu should report `skip=`, not `fail=`. |
| `Notify` | An on-screen info toast for the player, mirrored to the log so log-only sessions see it too. |
| `Lifecycle` | `WhenPlayerReady` — a coroutine that waits until the player is actually placed on a gameplay scene (past the game's void/staging coordinates) before your mod starts touching it, plus `IsSanePosition` for the same check inline. |
| `TableLoader<T>` / `EmbeddedRes` | Embedded-default-plus-config-override data tables: ship a sane default table inside your DLL, and let players override it by dropping a same-format file into `BepInEx/config/`. |
| `Keybinds` | A cross-mod keybind registry: each mod claims its keys, and if two mods claim the same key ForgeKit logs a warning naming both claimants and where to rebind. `IsFree(combo)` asks whether a candidate key is already claimed (useful when picking a default), and ForgeKit logs the full census once on the first frame of every boot, so any log pull carries it. |
| `NameCandidates` | The "config carries names, the game carries the truth" ladder. Asset identifiers (status names, item names) can't be known offline, so a mod ships a comma-separated candidate list in config and this resolves it against the running game: first candidate the registry knows wins, cached on the raw config string so a config reload re-resolves. A list that resolves to nothing warns **once**, naming every candidate. Two modes: `Resolve()` when your target is a prefab you apply by name, and `TryValidate()` + `FindFirst()` when the live object is reachable — that second mode treats the registry as a typo check only and asks live state every call, which is the safer default when two variants of a name can both exist. A failed lookup is never cached, so a name registered late still resolves. |
| `CommonVerbs` | A shared pack of generic dev verbs (item spawning, world/combat staging, engine diagnostics) any mod can register onto its own command channel. |

## The command channel

The command channel is the in-game iteration loop. A consuming mod constructs a `CommandChannel`
pointed at a file name (e.g. `YourMod_cmd.txt`) and calls `Tick()` from its `Update`. Writing a line
into `BepInEx/config/YourMod_cmd.txt` runs that verb on the next poll:

- The poll cadence is on **unscaled time**, so verbs fire even while the game is paused (a menu open,
  the game at `timeScale 0`).
- A command file may batch multiple lines — each non-blank, non-`#` line runs in order.
- A verb that throws is caught and logged, so one bad line in a batch doesn't kill the rest.
- Unknown verb, empty input, or the literal `help` prints the full registered verb list with help
  text.
- Every channel registers `script`, `scriptcancel` and `scriptstatus` itself, so any consumer can run
  a timed sequence of its own verbs from one file write.

Keybinds can drive the same verbs directly (`channel.Run("somedump")`), so a mod's F-keys and its
command file share one code path.

## The shared verb pack (CommonVerbs)

`CommonVerbs.RegisterAll(host, log)` adds a batch of generic dev verbs to a mod's own command channel
(they answer on that mod's `<mod>_cmd.txt`). Verbs are grouped into domains a consumer can toggle or
opt out of individually.

| Domain | Verbs |
|---|---|
| Items | `give`, `drop`, `useitem`, `givewater`, `equip`, `unequip` |
| World | `teleport`, `walkto`, `standoff`, `goto`, `settime`, `givemoney`, `face`, `moveto`, `firecamp` |
| Combat | `sethp`, `combatclear`, `killnearest`, `swing`, `lockon`, `lockoff` |
| Skills | `learnskill`, `unlearnskill`, `resetcooldowns`, `castspell` |
| Status | `grantstatus`, `removestatus` |
| Engine diagnostics | `statusdump`, `pos`, `scenedump`, `skydump`, `groundprobe`, `combatmgrdump`, `keybinds`, `ragdolldump`, `psdump` |
| Containers | `containerdump`, `containerroll` |
| Cheats | `cheats`, `cheatmenu`, `godmode`, `enemygod`, `needs`, `speedmult`, `timejump`, `debugfile` |
| Resilience | `unstick` (alias `unwedge`) |
| Config | `reloadcfg`, `set`, `cfgdump`, `cfgdrift` (only when the consuming mod hands the pack its config file — see below) |

A few highlights:

- `give [pouch\|bag\|ground] [qty] <name-or-ItemID>` / `drop [qty] <name-or-ItemID>` — spawn any item
  by name or numeric ItemID; an unresolved name logs did-you-mean suggestions.
- `givewater [clean\|river\|salt\|rancid\|leyline\|sparkling\|healing]` — spawn a pre-filled Waterskin.
- `goto <scene> [spawnPoint]` / `teleport <x> <y> <z>` / `settime <hour>` — world staging. `goto` and
  `settime` are host-only (a guest driving them would desync a co-op party).
- `firecamp [on\|off\|remove\|status] [distance 0.5-20]` — place a **real vanilla campfire** ahead of
  the player and **light** it, so a test can sample genuine ambient temperature. `off` extinguishes
  it where it stands (so recovery can be watched), `remove` destroys it, `status` prints the
  campfire's live `TemperatureSource` distance bands and the step predicted at your range. Host-only.

  This verb exists because `useitem Campfire Kit` cannot work, and that is worth knowing before
  reaching for it on any other deployable: a deployable's `Use` does **not** deploy anything, it
  opens an interactive placement mode (`BasicDeployable.OnItemUse` → `DeployablePlacer.StartPlacement`
  → `Character.StartDeploy`). The placer then idles until `Character.DeployInput` is driven by a real
  keypress; only on confirm does an RPC plus a `SetupGround` cast animation reach
  `Deployable.DeployableCast`, the method that instantiates the campfire. A command channel has no
  input frames to give, so `useitem` honestly reports `TryUse -> True` and nothing lands. `firecamp`
  skips the aim mode and does `DeployableCast`'s own work.

  **Lighting is not cosmetic.** `FueledContainer.StartInit` disables the campfire's
  `TemperatureSource`, and only `Kindle()` re-enables it; both of the game's heat-source searches
  skip a source that is not `isActiveAndEnabled`. An unlit campfire radiates exactly nothing, so a
  spawn-only verb would have produced a convincing false negative.
- **`standoff <metres> [bearing=<degrees>] [target=pet|nearest|<species>]`** and
  **`walkto <x> <z>`** — the **ground-safe** placement pair. `teleport` writes whatever height it is
  handed; these two take their height from the **navmesh** and **refuse to move you at all** when
  there is none. (`walkto` was defective in 0.4.5/0.4.6 — see the version note below.)
  See [Ground-safe placement](#ground-safe-placement-standoff--walkto) below.
- `learnskill <name-or-ItemID>` / `unlearnskill <name-or-ItemID\|all>` — teach or forget any skill,
  so a test save doesn't need a cheat menu. `castspell <name-or-ItemID>` then casts a *learned* skill
  through the real quickslot pipeline (requirements, cooldown and mana all apply for real).
- `moveto <nearest\|species> [gap]` / `face <…>` / `lockon` / `swing` — stage and land a real attack
  on a real creature from the command file, without a hand on the keyboard.
- `grantstatus <status-name> [target=player\|pet] [force]` / `removestatus` — apply or clear a status
  the vanilla way. `target=pet` only answers on a channel whose mod supplied a pet accessor; elsewhere
  it says so rather than silently acting on the player.
- The Cheats domain drives **the game's own hidden debug mode** (the wiki's "Debug Mode": a
  `DEBUG.txt` file arms it at boot, unlocking F1–F4 cheat windows). ForgeKit adds no cheat logic of
  its own — each verb calls the same code the vanilla F2 window's widgets call, and persists the
  same way, so verb-set and menu-set state always agree. Bare `cheats` prints the **full cheat-state
  audit** (gate, `DEBUG.txt`, every per-character flag, persisted values) — run it before diagnosing
  any "vanilla broke" report. `cheats on|off` flips the gate live (no file, no relaunch);
  `debugfile on|off` makes it persist across relaunches; `cheatmenu <items|cheats|skills|quests|hide>`
  opens the actual vanilla windows without their F-keys. `godmode`/`enemygod`/`needs off`/
  `speedmult`/`timejump <hours>` cover the F2 window's main toggles headlessly. Co-op notes:
  `godmode` replicates room-wide (the vanilla path is a Photon RPC to all), `enemygod` is local,
  `timejump` is host-only. ⚠ Once debug mode is armed, the *game's* debug hotkeys are live too, and
  they are invisible to the keybind registry: game F12 (screenshot) collides with Beastwhispering's
  F12 diag, game F3 (skills window) with AggroKit's F3 dump — the `cheatmenu` and `screenshot`
  verbs are the collision-free path.
- `keybinds` — report every key claimed by every mod in the family, grouped by key, with conflicts
  flagged (see the keybind registry below).
- `unstick` (alias `unwedge`) — the last-resort verb for a session that has stopped responding: bare
  it dumps the load-gate/pause/timescale state and changes nothing, `unstick fix` applies the
  smallest repair that fits, and `unstick force <step>` overrides a refusal you have read and
  accepted. The rungs, in the order `unstick fix` tries them:
  `prologue` → `gate` → `pausemenu` → `timescale` → `forceunpause` (and `forceready`, which is
  reachable only by naming it).

  **`prologue` is the first rung and the usual answer.** Outward shows "context screens" (the
  `ProloguePanel`) on a new game, on some area events and after a camp event. Showing it pushes a
  `Prologue` entry onto the loader's pause stack, and the ONLY thing that takes it off again is
  walking past the last screen with a keypress — which requires the game window to have OS focus.
  On an unattended or unfocused box that press never comes, so the panel stays up, the world sim
  stays paused, and the level-load coroutine stays parked *before* its "press any key" gate. Every
  other symptom follows from that: no AI detection, no wander, no aggro, no hunger, no temperature,
  no regen — while the mod command channel keeps answering normally, because it polls unscaled
  time by design. The `prologue` rung presses the key for you, up to three times in case a further
  context screen appears behind the first.

  Two things the dump tells you that it did not used to. `prologue: panelUp=True` on its own line
  names the holder; and after any rung, a **deferred** `verify` line re-reads the state three
  quarters of a second later and grades it `HELD`, `REVERTED` or `NO-CHANGE`. The immediate
  re-dump printed in the same frame as a repair can be stale — the loader only settles its pause
  state in its own `Update` — so the `verify` line, not the `applied` line, is the one that says
  whether gameplay is actually running again.
- `reloadcfg` — re-read that mod's `.cfg` from disk. BepInEx 5 has **no config file-watcher**, so a
  hand-edited config does nothing until this verb runs (or the game relaunches). It registers only
  when the mod supplies its config file, because `BaseUnityPlugin.Config` is `protected` — only code
  inside the plugin class can reach it:

  ```csharp
  CommonVerbs.RegisterAll(verbs, Logger, new CommonVerbsOptions
  {
      ConfigSource = () => Config,             // enables `reloadcfg`
      OnConfigReloaded = ApplySettings,        // optional: re-apply after the re-read
  });
  ```

  A mod whose settings need more than a re-read (rebuilding caches, re-stamping live objects) either
  passes `OnConfigReloaded` or keeps its own richer `reloadcfg` and excludes the pack's copy.

Each mod chooses which domains it takes; a mod that ships its own richer version of a verb excludes
the pack's copy and keeps its own. `RegisterAll` logs one line naming the domains it enabled and the
exclusions it honored, and warns about any exclusion that matched no verb — a typo in an exclusion
would otherwise silently leave the pack's copy registered.

Who takes the pack today: Beastwhispering (excludes `sethp` and `groundprobe`, keeping its
pet-aware supersets), Hireling, SkillKit, SpawnKit (the full pack) and HelloOutward (the scaffold
template, so a new mod inherits it). AggroKit deliberately takes only the command channel, not
the pack. Registries are per mod, so two mods owning the same verb name never collide; the pack's
log tags are a byte-stable grep contract. Sibling packs one layer up live where their failure
modes live: `SkillKit.SkillVerbs` (castdump/castclear/skillverify/skilldump/skillitemdump) and
`DonorKit.DonorVerbs` (photondump/audiodump/audioprune/terraindump/terrainfix, also answering on
`ck_cmd.txt`).

### Verb homing rule

Where a dev verb lives is decided by two rules:

- **ForgeKit owns engine-generic verbs** any mod might need; a kit owns diagnostics/repairs for
  failure modes *its own mechanism* causes. That is why `terraindump`/`terrainfix` live in DonorKit
  (donor-harvest terrain damage) while `skydump`/`psdump`/`ragdolldump`/`combatmgrdump` live here.
- **Vanilla-registry staging (skills/status) stays in ForgeKit** by the A10 2026-08-15 ruling:
  SkillKit owns custom-skill/status *authoring*, not the vanilla registries, and a SkillKit
  dependency is SideLoader-poisoned for CI-buildable consumers — see the `SkillVerbs.cs` /
  `StatusVerbs.cs` headers.

### Config drift at boot (`[CFGSKEW]` and `cfgdrift`)

A test session once spent an evening measuring a feature against a config key that had quietly
drifted from the value the build ships (`[Synergy] MaxStacks` was `4` on that install and `8` in the
defaults) — a hand edit, a live `set`, or an old `.cfg` left in place will all do it, and nothing in
the log said so. ForgeKit now sweeps every installed mod's own config file
(`BepInEx/config/<guid>.cfg`) on the first frame, next to the `[CONTRACT]`/`[STAMP]` boot-health
lines, and writes **one warning line per mod whose live values differ from the shipped defaults** —
and nothing at all when they match, so the line's presence is the whole signal:

```
[CFGSKEW] Beastwhispering: 2 of 84 entries differ from shipped defaults — Synergy.MaxStacks=4 (default 8), Diag.LogLevel=Trace (default Verbose)
```

It names the first 8 drifted keys and then says `…and N more`. To read the rest — or to check drift
at any point in a session, after a `set` as easily as at boot — run `cfgdrift [section]` on that
mod's command channel: the same rows, the same comparison, only the entries that differ (`cfgdump`
still lists everything and marks the drifted ones with `*default`). Both share one helper, so they
cannot disagree. The comparison is live-value-vs-shipped-default, not file-vs-build: a key the
running build no longer binds is invisible here, and `[STAMP]` remains the signal for a mixed install.

```
# BepInEx/config/cobalt.beastwhispering.cfg — the drift the line above is reporting
[Synergy]
MaxStacks = 4          # shipped default: 8
```

## Settings

ForgeKit has no configuration file of its own — it provides the command channel and config-table
helpers that consuming mods use. The one setting it *defines* lives in each consumer's own file:
`[Diag] LogLevel` in `BepInEx/config/<guid>.cfg` (see "Turning the logging up (or down)" above). The pieces it ships (`TableLoader`, `CommandChannel`, `Keybinds`)
read the *consuming* mod's files and config, always under that mod's logger. Each consumer gets its
own command file at `BepInEx/config/<Mod>_cmd.txt`; ForgeKit itself neither writes a `.cfg` nor polls
a command file.

## Commands

ForgeKit runs no command channel itself — it provides the channel machinery other mods use, and the
`CommonVerbs` pack they register onto their own channels. See the tables above.

## For modders

### Depend on it

Reference the kit and declare the dependency so BepInEx loads it first:

```xml
<!-- your .csproj -->
<ProjectReference Include="path\to\ForgeKit\ForgeKit.csproj" Private="false" />
```

`Private="false"` matters — it stops MSBuild from copying a second `ForgeKit.dll` into your mod's
output folder. The kit ships from its own `BepInEx/plugins/ForgeKit/` folder; your mod just
references it and declares the dependency.

```csharp
[BepInDependency(ForgeKit.Plugin.GUID, ForgeKit.Plugin.VERSION)]   // VERSION = min-version floor
public class Plugin : BaseUnityPlugin
{
    internal void Awake()
    {
        // Tell ForgeKit which kit version you were compiled against (the const inlines at YOUR build);
        // it logs a [CONTRACT] verdict at boot and warns on screen if the install mixes builds.
        ForgeKit.KitContract.Declare(NAME, ForgeKit.Plugin.GUID, ForgeKit.Plugin.VERSION);
        …
    }
}
```

Do the same for every kit you depend on. Why, and what the `[CONTRACT]`/`[STAMP]` lines mean:
[Kit versioning](versioning.md).

### Wire up a command channel and a verb

```csharp
[BepInPlugin(GUID, NAME, VERSION)]
[BepInDependency(ForgeKit.Plugin.GUID)]
public class Plugin : BaseUnityPlugin
{
    private CommandRegistry _commands;
    private CommandChannel _channel;

    void Awake()
    {
        _commands = new CommandRegistry(Logger);
        _commands.Register("hello", "hello — logs a greeting.", args => Logger.LogMessage("Hi!"));

        // Optional: add the shared dev-verb pack, resolving the local player yourself.
        CommonVerbs.RegisterAll(_commands, Logger, () => CharacterManager.Instance?.GetFirstLocalCharacter());

        _channel = new CommandChannel("YourMod_cmd.txt", Logger, _commands);
    }

    void Update() => _channel.Tick();
}
```

Everything takes the *consumer's* `ManualLogSource`, so every line lands under your mod's name in the
log.

### The command channel as an automation surface

The same file pipe described above for players is, from a modder's side, a ready-made hook for
**agentic or scripted testing** — an AI agent, a CI job, or any external script can drive the game
the same way a human editing `<Mod>_cmd.txt` would: write a verb, poll the log for the result, repeat.
Nothing special has to be built for this — it's the ordinary command channel, used programmatically
instead of by hand. This family's own live-test workflow works exactly this way: a test-runner writes
verbs into the channel and reads `LogOutput.log` for `[TAG]`-prefixed results, with no human at the
keyboard. Every `CommandChannel` also registers `script` / `scriptcancel` / `scriptstatus` on its own
— a whole timed sequence of verbs (`step ; wait 1s ; step ; ...`) runs from a single file write,
which matters most when the write itself is expensive (relayed over a network to a remote or guest
machine, say). If your mod's verbs are meant to be automation-friendly, keep
their output greppable (one clearly-tagged log line per outcome) — that log line *is* the automation's
return value.

### Claim your keybinds

A mod cannot see another mod's config, so a key collision is only knowable in a shared place. Each mod
claims its keys at boot; ForgeKit reports a clash naming both claimants and where to rebind:

```csharp
Keybinds.Claim(NAME, "tame the targeted animal", MyConfig.TameKey);   // ConfigEntry<KeyboardShortcut>
```

Re-claiming (a live rebind, a config reload) replaces the old claim, so retuning keys never spawns
phantom conflicts. The `keybinds` verb dumps the whole registry on demand.

### Gate your log volume (`ModLog`)

Swap the type of your plugin's static logger and bind the tier in `Awake`:

```csharp
internal static ModLog Log;          // was: ManualLogSource

internal void Awake()
{
    Log = ModLog.Bind(this, Logger); // binds [Diag] LogLevel on YOUR config file
}
```

`ModLog` carries the same method names as `ManualLogSource` and converts implicitly back to the raw
sink, so **existing call sites compile unchanged** and anything still taking a `ManualLogSource`
(`CommandRegistry`, `TableLoader`, `Notify`) keeps working. Re-tiering a noisy line is then just
renaming the method:

| Call | Emits at | Use for |
|---|---|---|
| `LogWarning` / `LogError` | always, ungated | faults. Never gate these. |
| `LogMessage` | `Normal` and up | boot lines, notices, self-test results — what a bug report needs |
| `LogInfo` | `Verbose` and up | normal per-feature diagnostics |
| `LogDebug` | `Trace` only | per-tick / per-frame / per-item detail |

For a hot path, check `Log.TraceOn` / `Log.VerboseOn` *before* building the interpolated string —
BepInEx evaluates the argument at the call site regardless of whether anything will write it.

Two traps worth knowing:

- **BepInEx filters at the SINK, not the source.** `ManualLogSource.Log` always raises the event and
  the disk/console listeners drop it by bit-test. That is why the gate is here and not in
  `BepInEx.cfg`: a sink filter still pays every `$"..."`, and it is one knob for every mod at once.
- **Re-tiering must not move the bytes.** Log tags are a grep contract for testplans and the remote
  log stream, so a demoted line keeps its exact text — only *whether* it is written changes. Dev
  machines pin `LogLevel = Trace` via `config/shared/*.cfg.overlay` so those greps keep matching.

### Extension points and traps

- **`CommandChannel` cadence.** The constructor takes `pollSeconds`, an `allLines` flag (run every
  line vs. the first non-blank line only), and `primeStamp` (skip commands already sitting in the file
  at boot). Defaults match the common case; the flags exist to preserve each consumer's historical
  behavior.
- **`Lifecycle.WhenPlayerReady`** must be started via your plugin's `StartCoroutine` (Unity coroutines
  need a MonoBehaviour host). Pass a stable `waitKey` (e.g. `this`) to auto-supersede an older
  in-flight wait when two scene loads land in quick succession — otherwise a door-to-door transition
  could double-fire your per-scene setup.
- **`TableLoader<T>` / `EmbeddedRes`** load resources from the *consumer's* assembly, not ForgeKit's —
  pass your own `Assembly` so the loader finds your embedded default. A missing default is always
  logged, never silent.
- **`Notify.Log`** is a settable static; assign it (`Notify.Log = Logger`) in your `Awake` so toasts
  are mirrored under your mod.

### What stays OUT of the kit (the extraction rule)

ForgeKit was built (2026-07-07) when the third copy of the command channel appeared — that was the
agreed trigger for lifting shared dev tooling into a kit. The rule that governs what gets lifted
next:

- **Not for the kit** (companion-specific — extracting would force it): `BodyFactory` /
  `CompanionBody` / `Pet` / `PetSaveStore`, and the pet compute layer (loyalty, temperature, etc.).
- The brain-strip / inactive-holder clone is a clever Outward *recipe*, not an API: it is
  documented in the repo's `education/` notes, not turned into kit surface.
- The compute/test split (`core/` netstandard2.0 + `tests/` xUnit, zero game boots) and the
  deploy/log scripts (`scripts/`) stay **conventions**, not kit code.
- When something else proves reusable, **first make it pet-free and isolated inside its consuming
  mod, then lift it** — never extract straight from a tangled call site.

## See also

- [Kits index](./README.md)
- [Installing](../installing.md)
- Kits that build on ForgeKit: [AggroKit](./aggrokit.md), [DonorKit](./donorkit.md),
  [EnchantKit](./enchantkit.md), [NetKit](./netkit.md), [SkillKit](./skillkit.md),
  [StoryKit](./storykit.md), [CompanionKit](./companionkit.md), [SpawnKit](./spawnkit.md)
- [Wiki home](../README.md)


## Ground-safe placement (`standoff` / `walkto`)

`teleport <x> <y> <z>` is a **raw** write: the height you type is the height you get, with no ground
probe of any kind. That is deliberate — reproducing a coordinate triple from an earlier log has to
give back exactly that triple — but it means a guessed height can drop the character into a long
fall. Two verbs exist so a script never has to guess.

### `standoff <metres> [bearing=<degrees>] [target=pet|nearest|<species>]`

Places you a **measured distance** from a reference body, on ground that body can walk back over,
facing it. This is the primitive for anything phrased "stand N metres away and watch what the
companion does" — leash distances, recall ranges, aggro radii.

```
standoff 42                        # 42 m from your pet, keeping the side you are already on
standoff 84 bearing=180            # 84 m, due south of it
standoff 12 target=nearest         # 12 m from the nearest wild creature
standoff 20 target=Wolf Hyena      # …from a named species
```

- **Distance first, keywords after.** `bearing=` and `target=` may come in either order; anything
  else on the line is a parse error rather than a silent ignore.
- **`target=pet` needs a pet-aware channel.** It reads the same consumer-supplied accessor that
  `grantstatus target=pet` uses, so it answers on Beastwhispering's channel and says so plainly
  elsewhere. `target=nearest` / `target=<species>` work anywhere.
- **Distance is clamped** to 1–200 m, and the clamp is logged.
- **How it searches.** A fan of bearings (straight, ±20°, ±40°, ±60°, ±90°, ±135°, 180°) across a
  ladder of ranges, tried **range-major**: every bearing at the exact distance you asked for is
  probed before any shorter one, because the distance is usually the thing under test. Each
  candidate is navmesh-sampled, settled onto the real collider surface, height-sanity-checked
  against the reference body, and then **path-checked** from the reference body's own navmesh
  polygon — a spot across a canyon is real ground but a useless place to measure a leash from.
- **Failure is loud and total.** No landing found ⇒ nothing moves, an on-screen toast, and one
  `[STANDOFF] REFUSED` line quoting the search radius, tolerance and candidate count. A landing that
  is on ground but **not** path-connected is used with a prominent warning saying the reading will
  measure an island rather than a leash.

### `walkto <x> <z>`

Absolute placement with **no height argument** — the **navmesh** decides it.

```
walkto 1250.5 -430
```

- **Navmesh or nothing.** The only thing that may set the height is a `NavMesh.SamplePosition` hit
  (settled onto the collider surface underneath it). If the requested x/z has no navmesh anywhere in
  its column, the verb **refuses** and nothing moves — it does not fall back to a bare collider.
- **It sweeps the column, not one height.** Sample centres are tried downward-first from where you
  currently stand, so a call made from an already-wrong elevation can still find the mesh below it
  rather than compounding the error.
- **It will not lift you.** A landing more than 3 m above your current height is only ever taken when
  no lower rung of the column had navmesh at all, and then it is announced in the log before the
  write, so a following distance measurement can be discarded.
- **Failure is loud and total.** One `[WALKTO] REFUSED` line naming the probe heights, plus a
  diagnostic note about what collider (if any) is in that column — reported, never used — and an
  on-screen toast.

> **Version note.** In ForgeKit **0.4.5 and 0.4.6** `walkto` had a collider fallback that could not
> tell a floor from a ceiling: in a navmesh-free area it placed the character on the first collider a
> long downward ray met, which in one measured case was **23.9 m above the ground on the character's
> own x/z**, and repeat calls re-cast from the new height and could never come back down. Do not use
> `walkto` on those two builds; **0.4.7** replaces the fallback with a refusal.

### Which one to use

| You want | Verb |
|---|---|
| A measured distance from your companion or a creature | `standoff` |
| A known x/z landmark, ground height unknown (and on navmesh) | `walkto` |
| To reproduce an exact coordinate triple from a log, height included | `teleport` |

`teleport` still does what it always did. It now also **warns before it writes** when the
destination is more than 3 m above the nearest ground, and names these two verbs in that warning.

Log tags: `[STANDOFF]`, `[WALKTO]`, and `[DEV]` for `teleport`.
