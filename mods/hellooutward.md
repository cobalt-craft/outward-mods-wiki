# HelloOutward — the starter mod to copy

**HelloOutward** is a minimal example mod for Outward that does almost nothing on its own — it exists
to be copied. It is the template every other mod in this family is forked from, wired up with the
shared dev tooling so a new mod builds, deploys, and has a working in-game command loop from the
first line you write. It's for modders; there's nothing here to play with.

**At a glance**
- Type: gameplay mod (starter template)
- GUID: `cobalt.hellooutward`
- Requires: BepInEx 5 (Outward's Mono branch), [ForgeKit](../kits/forgekit.md)
- Config: `BepInEx/config/cobalt.hellooutward.cfg`
- Commands: `BepInEx/config/HelloOutward_cmd.txt`

## For players

There's nothing to play. HelloOutward logs a greeting when it loads and again on each area load; it's
a scaffold for building your own mod, not a feature.

## What you get out of the box

Copy the template and you inherit, already wired:

- A **[ForgeKit](../kits/forgekit.md) command channel** — write a verb into
  `BepInEx/config/HelloOutward_cmd.txt` and it runs on the next poll. `help` lists everything;
  `selftest` runs the template's `[SELFTEST]` report. The poll is on unscaled time, so verbs fire even
  while the game is paused.
- **Config binding** — one example `Config.Bind(...)` entry, so the mod generates its own
  `BepInEx/config/<guid>.cfg`.
- The **keybind Claim pattern** — the copy-ready (commented) pair for binding a key *and* registering
  it with ForgeKit's cross-mod keybind registry, plus a self-test assert that fails loudly if two mods
  claim the same key.
- A **smoke-test Harmony patch** that confirms Harmony and the game-library references resolve at
  runtime.

## Scaffold a new mod

`scripts/new-mod.sh <ModName>` clones HelloOutward into a ready-to-build project. It:

- copies `src/HelloOutward/` to `src/<ModName>/` (dropping any old `bin`/`obj`),
- renames `HelloOutward.csproj` to `<ModName>.csproj`,
- rewrites the identifiers in every `.cs`/`.csproj` — GUID `cobalt.hellooutward` → `cobalt.<modname>`
  (lowercased), and both `HelloOutward` and `Hello Outward` → `<ModName>` (so the namespace,
  `AssemblyName`, class names, `NAME`, and the command-channel filename all follow), and
- adds the new project to the solution (`dotnet sln Outward.slnx add ...`).

`<ModName>` must be PascalCase letters and digits only — no spaces or dots.

```sh
scripts/new-mod.sh CampfireTweaks
```

Then:

1. Edit `NAME` / `VERSION` in `src/<ModName>/Plugin.cs`.
2. `dotnet build Outward.slnx -c Release` (output lands in `dist/<ModName>/`).
3. Deploy it to the game machine — set your destination once, then `./scripts/deploy.sh`:

   ```sh
   echo 'user@gamebox:/path/to/Outward/BepInEx/plugins' > .deploy-target
   dotnet build Outward.slnx -c Release
   ./scripts/deploy.sh
   ```

The new mod inherits ForgeKit's dev loop from the start: write verbs into
`BepInEx/config/<ModName>_cmd.txt` (`help` lists them, `selftest` runs the `[SELFTEST]` template).

If you'd rather not use the script, the manual steps are the same: `cp -r src/HelloOutward
src/MyMod`, rename the `.csproj` and its `AssemblyName`, change the GUID/NAME/namespace in
`Plugin.cs`, and `dotnet sln Outward.slnx add src/MyMod/MyMod.csproj`.

## The minimal Plugin.cs

The whole shape of a mod fits in one class. The essentials:

```csharp
using BepInEx;
using BepInEx.Configuration;
using BepInEx.Logging;
using HarmonyLib;
using ForgeKit;

namespace HelloOutward
{
    [BepInPlugin(GUID, NAME, VERSION)]
    [BepInDependency(ForgeKit.Plugin.GUID, BepInDependency.DependencyFlags.HardDependency)]
    public class Plugin : BaseUnityPlugin
    {
        public const string GUID = "cobalt.hellooutward";   // change these when you fork
        public const string NAME = "Hello Outward";
        public const string VERSION = "1.0.0";

        internal static ManualLogSource Log;
        public static ConfigEntry<bool> GreetOnEachAreaLoad;

        private CommandRegistry _commands;
        private VerbHost _verbs;
        private CommandChannel _channel;

        internal void Awake()
        {
            Log = Logger;

            GreetOnEachAreaLoad = Config.Bind(
                "General", "GreetOnEachAreaLoad", true,
                "Log a greeting every time the game loads prefab resources.");

            RegisterVerbs();
            _channel = new CommandChannel("HelloOutward_cmd.txt", Log, _commands);

            new Harmony(GUID).PatchAll();   // applies every [HarmonyPatch] in this assembly
        }

        internal void Update() => _channel.Tick();

        private static Character LocalPlayer => CharacterManager.Instance?.GetFirstLocalCharacter();

        private void RegisterVerbs()
        {
            _commands = new CommandRegistry(Log);
            _verbs = new VerbHost(_commands, Log, () => LocalPlayer);

            _verbs.Register("selftest", "Run the template self-test.",
                _ => SelfTest(), tag: "[HelloOutward]", needsPlayer: false);
        }
    }
}
```

Notes on the pieces:

- **`[BepInDependency]` on ForgeKit** makes BepInEx load ForgeKit first — without it your command
  channel might construct before the kit is ready.
- **Verbs register through `VerbHost`, not the raw registry.** `VerbHost` wraps each verb with a
  shared prologue: a local-player guard (a player-guarded verb body never has to null-check
  `ctx.Player`), an optional `masterOnly:true` Photon gate for anything that mutates networked state,
  and a tagged try/catch. `needsPlayer:false` opts a verb out of the player guard (as `selftest` does,
  so it runs at the main menu). `ctx.Arg(n)` / `ctx.Tail()` read arguments.
- **`_channel.Tick()` in `Update`** is what polls the command file each frame.

### Adding a keybind

Keys are a cross-mod resource — a mod can't see another mod's config, so
[ForgeKit's `Keybinds`](../kits/forgekit.md) registry is the only place a collision is knowable. Bind
your key *and* Claim it:

```csharp
GreetKey = Config.Bind("Keys", "GreetKey",
    new KeyboardShortcut(KeyCode.None), "Say hello.");
ForgeKit.Keybinds.Claim(NAME, "say hello", GreetKey);
```

The template ships with this pair commented out and bound to `KeyCode.None` on purpose, so a fresh
fork can't instantly collide — pick a genuinely free key when you wire a real bind. Keep the
`!Keybinds.HasConflicts()` check in the self-test; it's what makes a clash fail loudly at boot or when
you run `selftest`.

### Growing beyond the template

- Add a `core/<Mod>.Core` (netstandard2.0) project plus a `ProjectReference` if the mod grows pure
  logic worth unit-testing off the game.
- Add a SideLoader reference only if the mod registers custom SideLoader content (items, recipes,
  skills).
- Everything else — build settings, game references — is inherited from the root
  `Directory.Build.props`, so the `.csproj` stays a few lines.

## See also

- [ForgeKit](../kits/forgekit.md) — the dev tooling HelloOutward inherits (command channel, self-test
  harness, keybind registry, shared verb pack)
- [Kits index](../kits/README.md)
- [Mods index](./README.md)
- [Wiki home](../README.md)
