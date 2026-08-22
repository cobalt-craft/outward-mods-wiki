# Installing

How to get these mods running in **Outward: Definitive Edition**.

**At a glance**
- Outward must be on its **Mono** Steam branch (not the default IL2CPP build).
- Mods are loaded by **BepInEx 5**.
- Each mod is a folder under `BepInEx/plugins/`, alongside the kits it depends on.
- Settings live in `BepInEx/config/cobalt.<mod>.cfg` — see [Where settings live](#where-settings-live).

## 1. Put Outward on the Mono branch (required)

In Steam: **Outward → Properties → Betas → select `mono`**, and let it update.

The whole mod stack is built for Outward's **Mono** runtime. On the default **IL2CPP** build the mods
silently do nothing — the game just runs vanilla, with no error and no log. This is the single most
common reason "nothing loads".

You can tell the two apart in the game's install folder (`Outward_Defed/`):

| Build | Tell-tale files |
|---|---|
| **Mono** (correct) | `Outward Definitive Edition_Data/Managed/Assembly-CSharp.dll`, a `MonoBleedingEdge/` folder |
| **IL2CPP** (wrong) | `GameAssembly.dll`, an `Outward Definitive Edition_Data/il2cpp_data/` folder |

## 2. Install BepInEx

Install the **BepInEx 5** pack for Outward (available on
[Thunderstore](https://outward.thunderstore.io/)). Launch the game once so BepInEx generates its
`BepInEx/` folder, then quit.

Keep the whole loader overlay together in the game root: `BepInEx/`, `winhttp.dll`,
`doorstop_config.ini` **and `mono_fix/`**. `mono_fix/` holds a patched `mscorlib`, and without it the
preloader crashes before any mod loads (it writes a `preloader_*.log` beside the game executable).

**Linux / Proton / Steam Deck:** set the Steam launch option

```
WINEDLLOVERRIDES="winhttp=n,b" %command%
```

and force a Proton compatibility tool — a native "Steam Linux Runtime" won't load BepInEx. Use the
bare `winhttp`, **without** a `.dll` suffix; some Proton versions ignore the suffixed form and the
game then runs vanilla.

Three Linux details that look wrong but aren't, and one diagnostic:

- A wrapper prefix in the same launch option (a gamemode/keyboard-remapper style
  `WINEDLLOVERRIDES="winhttp=n,b" some-wrapper %command%`) is harmless — BepInEx still loads.
- The game's working directory is the parent `…/steamapps/common/Outward`, **not** `Outward_Defed/`.
  That is normal for Outward DE; nothing needs fixing.
- A fresh Steam install defaults to the IL2CPP branch, and on it the whole stack fails *silently*:
  Doorstop's `winhttp` loads, finds no Mono runtime to hook, and the game runs vanilla with no
  `LogOutput.log` and no preloader crash. Check the branch (step 1) before anything else.
- To diagnose a BepInEx that never appears, launch once with `PROTON_LOG=1` in the launch option and
  grep `~/steam-<appid>.log` for `winhttp`, `GameAssembly` and `Doorstop` — it shows whether the DLL
  override took effect and which runtime the game actually started.

## 3. Install the mods

Three ways:

- **Mod manager** — if a mod is published to Thunderstore, install it there and its dependencies come
  along automatically.
- **The bundle installer** — if you were handed a zip of these mods, unzip it inside your
  `Outward_Defed/` folder and double-click `Install-Mods.bat`. See
  [Using the bundle installer](#using-the-bundle-installer).
- **By hand** — drop each mod's folder into `BepInEx/plugins/`, together with the kits it depends on
  (see the table below). If BepInEx logs a missing-dependency error and refuses to load a mod, one of
  its required kit folders isn't there.

### Dependencies

Every mod needs its kits present. A mod manager resolves these for you; installing by hand, add them
yourself. Each row lists the mod's **full** kit closure, not just its direct requirements:

| Install this mod | …and also these |
|---|---|
| [Beastwhispering](mods/beastwhispering/README.md) | ForgeKit, SkillKit, CompanionKit, DonorKit, AggroKit, NetKit, StoryKit, **SideLoader** |
| [DangerousRoads](mods/dangerous-roads.md) | ForgeKit, SpawnKit, CompanionKit, DonorKit, AggroKit, NetKit |
| [Hireling](mods/hireling.md) | ForgeKit, CompanionKit, DonorKit, AggroKit, NetKit |
| [Cloudward](mods/cloudward.md) | ForgeKit |
| [SpawnKit](kits/spawnkit.md) | ForgeKit, CompanionKit, DonorKit, AggroKit, NetKit |

(SideLoader is a separate community mod that Beastwhispering builds on for its custom items and
skills. It is a hard requirement: without it, Beastwhispering does not load at all.)

**One mod ships a file outside `plugins/`:** [Cloudward](mods/cloudward.md) includes
`Cloudward.Preload.dll`, which belongs in `BepInEx/patchers/`. In `plugins/` it is inert and silently
does nothing.

[HelloOutward](mods/hellooutward.md) is a template for modders rather than something to play; it
needs only ForgeKit.

### Using the bundle installer

The hand-off zip contains a `payload/` folder, `BUNDLE.json`, `install.ps1` and a double-click
`Install-Mods.bat`. Unzip the whole thing (keeping its folder structure) inside your Outward install
and run the `.bat`. It finds the game itself — from where the zip was unpacked, or by probing your
Steam libraries — or you can point it at the game explicitly.

What it does, and deliberately does not do:

- **It refuses outright on an IL2CPP install**, with instructions for switching branches. It also
  refuses while Outward is running.
- **The bundled mods are deleted and re-copied on every run**, so a stale DLL or an orphaned data file
  from an older build can never survive an install.
- **BepInEx and SideLoader are installed only if they are absent.** An existing copy is left alone, so
  the bundle can't overwrite or downgrade a setup you already have. `-Force` overrides that.
- **It never touches `BepInEx/config/` or `SaveGames/`** — on install or on uninstall. Your settings
  and your characters are yours.

| Option | What it does |
|---|---|
| `-DryRun` | Print exactly what would happen and change nothing. |
| `-Uninstall` | Remove the bundle's own mod folders, leaving BepInEx, SideLoader, settings and saves alone. |
| `-Force` | Overwrite BepInEx and SideLoader as well, instead of keeping what's already installed. |
| `-GameDir <path>` | Point at the `Outward_Defed` folder if it isn't found automatically. |

Run them from PowerShell, e.g. `.\install.ps1 -DryRun`.

## Where settings live

Each mod generates its own file on first launch:

```
BepInEx/config/cobalt.<mod>.cfg
```

for example `cobalt.beastwhispering.cfg`, `cobalt.hireling.cfg`, `cobalt.cloudward.cfg`. Every setting
is written out with its default and a description, so the generated file is its own documentation.
Each mod's wiki page lists its keys.

**Edit these only while the game is closed.** A launch rewrites them, so a change made with the game
running is lost.

```ini
## Example — BepInEx/config/cobalt.hireling.cfg
[Keys]
## Recruit the nearest recruitable NPC.
RecruitKey = F4

[Recruit]
## How close (m) an NPC must be to recruit it.
RecruitRange = 15
```

Most mods also read a plain-text command file, `BepInEx/config/<Mod>_cmd.txt`, used for diagnostics —
see each mod's *Commands* section.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No mods load at all, no crash log | Outward is on the IL2CPP branch | Switch to the Mono branch (step 1) |
| No mods load, and a `preloader_*.log` appeared | `mono_fix/` is missing from the game root | Reinstall the full BepInEx overlay (step 2) |
| BepInEx loads but one mod refuses | A required kit folder is missing | Add the kits from the table above |
| Nothing loads on Linux | Missing launch option / native runtime | Set the `WINEDLLOVERRIDES` option and force Proton (step 2) |
| Beastwhispering doesn't load | SideLoader isn't installed | Install SideLoader — it's a hard requirement |
| Cloudward never installs a fetched mod update | `Cloudward.Preload.dll` is in `plugins/` | Move it to `BepInEx/patchers/` |
| Game hangs (often a load deadlock) and won't die on Linux | `pkill -x wine64-preloader` / `pkill -f <appid>` miss it: the game process's comm is `Outward Definit` and its cmdline lacks the appid | `pgrep -f 'Outward Definitive Edition'`, pick the PID whose `/proc/<pid>/comm` is `Outward Definit` (a load deadlock shows state `S` on `futex_wait_multiple`), then `kill -9 <pid>`. Note `pgrep -f` also matches your own shell — use `pgrep -x 'Outward Definit'` when you only need a yes/no "is it running" |

The log to read is `BepInEx/LogOutput.log` in the game folder. If it doesn't exist at all, BepInEx
never loaded — that's step 1 or step 2, not the mods.

## See also

- [Wiki home](README.md)
- [Kits index](kits/README.md) — what each library is
- [Mods index](mods/README.md)
