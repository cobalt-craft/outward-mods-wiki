# Outward Mods — Wiki

Documentation for a family of mods and reusable libraries for **Outward: Definitive Edition**.

Two kinds of project live here:

- **Mods** — things you install and play with (tame animals, recruit followers, spawn creatures).
- **Kits** — reusable libraries that mods are built from. You don't install a kit on its own; a mod
  that needs one pulls it in.

New here? Start with **[Installing](installing.md)**.

## For players

| Mod | What it does |
|---|---|
| [Beastwhispering](mods/beastwhispering/README.md) | Tame wild Outward animals as persistent pets — feed them, bond with them, and fight alongside them. |
| [Hireling](mods/hireling.md) | Recruit a townsperson as a persistent human follower. |
| [SpawnKit](kits/spawnkit.md) | Spawn any creature beside you from an in-game menu. |

## For modders

Building an Outward mod? These libraries do the heavy lifting. The **[Kits index](kits/README.md)**
has the dependency map and a "which kit do I need" table.

| Kit | What it gives you |
|---|---|
| [ForgeKit](kits/forgekit.md) | Dev tooling: command channel, self-test, on-screen toasts, config tables, keybind registry. |
| [SkillKit](kits/skillkit.md) | Ship custom learnable skills. |
| [CompanionKit](kits/companionkit.md) | Persistent creature companions. |
| [AggroKit](kits/aggrokit.md) | Aggro and threat control. |
| [NetKit](kits/netkit.md) | Photon co-op networking. |
| [SpawnKit](kits/spawnkit.md) | Runtime creature spawning. |
| [StoryKit](kits/storykit.md) | NPCs, trainers, and dialogue. |
| [HelloOutward](mods/hellooutward.md) | Starter template for a new mod. |

## About this wiki

These pages are written to be read on GitHub — every link is relative, so they work in a clone or a
browser. Pages describe what each project does, not its internal build status. Contributing a page?
See the **[style guide](STYLE.md)**.
