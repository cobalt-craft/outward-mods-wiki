# Installing

How to get these mods running in **Outward: Definitive Edition**.

**At a glance**
- Outward must be on its **Mono** Steam branch (not the default IL2CPP build).
- Mods are loaded by **BepInEx 5**.
- Each mod is a folder under `BepInEx/plugins/`, alongside the kits it depends on.

## 1. Put Outward on the Mono branch (required)

In Steam: **Outward → Properties → Betas → select `mono`**, and let it update.

The whole mod stack is built for Outward's **Mono** runtime. On the default **IL2CPP** build the mods
silently do nothing — the game just runs vanilla, with no error and no log. This is the single most
common reason "nothing loads".

You can tell the two apart in the game's install folder:

| Build | Tell-tale files |
|---|---|
| **Mono** (correct) | `..._Data/Managed/Assembly-CSharp.dll`, a `MonoBleedingEdge/` folder |
| **IL2CPP** (wrong) | `GameAssembly.dll`, a `..._Data/il2cpp_data/` folder |

## 2. Install BepInEx

Install the **BepInEx 5** pack for Outward (available on
[Thunderstore](https://outward.thunderstore.io/)). Launch the game once so BepInEx generates its
`BepInEx/` folder, then quit.

**Linux / Proton / Steam Deck:** set the Steam launch option
`WINEDLLOVERRIDES="winhttp=n,b" %command%` and force a Proton compatibility tool (a native Steam Linux
Runtime won't load BepInEx).

## 3. Install the mods

Two ways:

- **Mod manager** — if a mod is published to Thunderstore, install it there and its dependencies come
  along automatically.
- **By hand** — drop each mod's folder into `BepInEx/plugins/`, together with the kits it depends on
  (see the table below). If BepInEx logs a missing-dependency error and refuses to load a mod, one of
  its required kit folders isn't there.

### Dependencies

Every mod needs its kits present. A mod manager resolves these for you; installing by hand, add them
yourself:

| Install this mod | …and also these |
|---|---|
| [Beastwhispering](mods/beastwhispering/README.md) | ForgeKit, SkillKit, CompanionKit, AggroKit, NetKit, StoryKit, **SideLoader** |
| [Hireling](mods/hireling.md) | ForgeKit, CompanionKit, AggroKit, NetKit |
| [Cloudward](mods/cloudward.md) | ForgeKit |
| [SpawnKit](kits/spawnkit.md) | ForgeKit, CompanionKit, AggroKit, NetKit |

(SideLoader is a separate community mod that Beastwhispering builds on for its custom items and
skills.)

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No mods load at all, no crash log | Outward is on the IL2CPP branch | Switch to the Mono branch (step 1) |
| BepInEx loads but one mod refuses | A required kit folder is missing | Add the kits from the table above |
| Nothing loads on Linux | Missing launch option / native runtime | Set the `WINEDLLOVERRIDES` option and force Proton (step 2) |

## See also

- [Wiki home](README.md)
- [Kits index](kits/README.md) — what each library is
