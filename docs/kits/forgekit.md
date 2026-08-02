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
| World | `teleport`, `goto`, `settime`, `givemoney`, `face`, `moveto` |
| Combat | `sethp`, `combatclear`, `killnearest`, `swing`, `lockon`, `lockoff` |
| Skills | `learnskill`, `unlearnskill`, `resetcooldowns`, `castspell` |
| Status | `grantstatus`, `removestatus` |
| Engine diagnostics | `statusdump`, `pos`, `scenedump`, `skydump`, `groundprobe`, `combatmgrdump`, `keybinds`, `ragdolldump`, `psdump` |
| Containers | `containerdump`, `containerroll` |
| Resilience | `unstick` (alias `unwedge`) |
| Config | `reloadcfg` (only when the consuming mod hands the pack its config file — see below) |

A few highlights:

- `give [pouch\|bag\|ground] [qty] <name-or-ItemID>` / `drop [qty] <name-or-ItemID>` — spawn any item
  by name or numeric ItemID; an unresolved name logs did-you-mean suggestions.
- `givewater [clean\|river\|salt\|rancid\|leyline\|sparkling\|healing]` — spawn a pre-filled Waterskin.
- `goto <scene> [spawnPoint]` / `teleport <x> <y> <z>` / `settime <hour>` — world staging. `goto` and
  `settime` are host-only (a guest driving them would desync a co-op party).
- `learnskill <name-or-ItemID>` / `unlearnskill <name-or-ItemID\|all>` — teach or forget any skill,
  so a test save doesn't need a cheat menu. `castspell <name-or-ItemID>` then casts a *learned* skill
  through the real quickslot pipeline (requirements, cooldown and mana all apply for real).
- `moveto <nearest\|species> [gap]` / `face <…>` / `lockon` / `swing` — stage and land a real attack
  on a real creature from the command file, without a hand on the keyboard.
- `grantstatus <status-name> [target=player\|pet] [force]` / `removestatus` — apply or clear a status
  the vanilla way. `target=pet` only answers on a channel whose mod supplied a pet accessor; elsewhere
  it says so rather than silently acting on the player.
- `keybinds` — report every key claimed by every mod in the family, grouped by key, with conflicts
  flagged (see the keybind registry below).
- `unstick` (alias `unwedge`) — the last-resort verb for a session that has stopped responding: bare
  it dumps the load-gate/pause/timescale state and changes nothing, `unstick fix` applies the
  smallest repair that fits, and `unstick force <step>` overrides a refusal you have read and
  accepted.
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
- Kits that build on ForgeKit: [AggroKit](./aggrokit.md), [DonorKit](./donorkit.md),
  [EnchantKit](./enchantkit.md), [NetKit](./netkit.md), [SkillKit](./skillkit.md),
  [StoryKit](./storykit.md), [CompanionKit](./companionkit.md), [SpawnKit](./spawnkit.md)
- [Wiki home](../README.md)
