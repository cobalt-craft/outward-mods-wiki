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

You won't interact with ForgeKit directly. It ships as a dependency of the mods you install and does
nothing on its own — install it because another mod declares it as a requirement.

## What's in it

| Piece | What it gives a modder |
|---|---|
| `CommandChannel` + `CommandRegistry` | A file-driven dev-command loop: write a verb into `BepInEx/config/<yourmod>_cmd.txt` and it runs on the next poll. The poll runs on unscaled time, so it still fires while the game is paused. Unknown verb (or `help`) prints every registered verb with its help text. |
| `VerbHost` / `VerbContext` | A thin wrapper over the registry that adds a per-verb prologue: resolve the local player, an optional Photon master-only gate, and a per-verb error tag. |
| `SelfTestHarness` | The `[SELFTEST] BEGIN` / `PASS:` / `FAIL:` / `DONE pass= fail=` report shape — wire up a handful of `Check(name, condition)` calls and get one consistent, greppable self-test report. |
| `Notify` | An on-screen info toast for the player, mirrored to the log so log-only sessions see it too. |
| `Lifecycle` | `WhenPlayerReady` — a coroutine that waits until the player is actually placed on a gameplay scene (past the game's void/staging coordinates) before your mod starts touching it, plus `IsSanePosition` for the same check inline. |
| `TableLoader<T>` / `EmbeddedRes` | Embedded-default-plus-config-override data tables: ship a sane default table inside your DLL, and let players override it by dropping a same-format file into `BepInEx/config/`. |
| `Keybinds` | A cross-mod keybind registry: each mod claims its keys, and if two mods claim the same key ForgeKit logs a warning naming both claimants and where to rebind. |
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

Keybinds can drive the same verbs directly (`channel.Run("somedump")`), so a mod's F-keys and its
command file share one code path.

## The shared verb pack (CommonVerbs)

`CommonVerbs.RegisterAll(host, log)` adds a batch of generic dev verbs to a mod's own command channel
(they answer on that mod's `<mod>_cmd.txt`). Verbs are grouped into domains a consumer can toggle or
opt out of individually.

| Domain | Verbs |
|---|---|
| Items | `give`, `drop`, `useitem`, `givewater`, `equip`, `unequip` |
| World | `teleport`, `goto`, `settime`, `givemoney` |
| Combat | `sethp`, `combatclear`, `killnearest` |
| Skills | `learnskill`, `unlearnskill`, `resetcooldowns` |
| Engine diagnostics | `statusdump`, `scenedump`, `skydump`, `groundprobe`, `combatmgrdump`, `keybinds`, `ragdolldump`, `psdump` |

A few highlights:

- `give [pouch\|bag\|ground] [qty] <name-or-ItemID>` / `drop [qty] <name-or-ItemID>` — spawn any item
  by name or numeric ItemID; an unresolved name logs did-you-mean suggestions.
- `givewater [clean\|river\|salt\|rancid\|leyline\|sparkling\|healing]` — spawn a pre-filled Waterskin.
- `goto <scene> [spawnPoint]` / `teleport <x> <y> <z>` / `settime <hour>` — world staging. `goto` and
  `settime` are host-only (a guest driving them would desync a co-op party).
- `learnskill <name-or-ItemID>` / `unlearnskill <name-or-ItemID\|all>` — teach or forget any skill,
  so a test save doesn't need a cheat menu.
- `keybinds` — report every key claimed by every mod in the family, grouped by key, with conflicts
  flagged (see the keybind registry below).

Each mod chooses which domains it takes; a mod that ships its own richer version of a verb excludes
the pack's copy and keeps its own.

## Settings

ForgeKit has no configuration file of its own — it provides the command channel and config-table
helpers that consuming mods use. The pieces it ships (`TableLoader`, `CommandChannel`, `Keybinds`)
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

### Claim your keybinds

A mod cannot see another mod's config, so a key collision is only knowable in a shared place. Each mod
claims its keys at boot; ForgeKit reports a clash naming both claimants and where to rebind:

```csharp
Keybinds.Claim(NAME, "tame the targeted animal", MyConfig.TameKey);   // ConfigEntry<KeyboardShortcut>
```

Re-claiming (a live rebind, a config reload) replaces the old claim, so retuning keys never spawns
phantom conflicts. The `keybinds` verb dumps the whole registry on demand.

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

## See also

- [Kits index](./README.md)
- [Installing](../installing.md)
- Kits that build on ForgeKit: [AggroKit](./aggrokit.md), [NetKit](./netkit.md),
  [SkillKit](./skillkit.md), [StoryKit](./storykit.md), [CompanionKit](./companionkit.md),
  [SpawnKit](./spawnkit.md)
- [Wiki home](../README.md)
