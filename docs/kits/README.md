# Kits

Reusable libraries for Outward mods. A **kit** isn't installed on its own — a mod that needs it
declares a dependency (`[BepInDependency]`) and BepInEx loads the kit first.

If you're a player, you don't interact with these directly; they arrive as dependencies of the mods
you install. This section is for **modders** building on them.

## Which kit do I need?

| Kit | Use it to… | Depends on | SideLoader |
|---|---|---|---|
| [ForgeKit](forgekit.md) | add dev commands, self-tests, on-screen toasts, config-backed data tables, and register keybinds | — | no |
| [AggroKit](aggrokit.md) | control what enemies target (force-target, calm, taunt) | ForgeKit | no |
| [NetKit](netkit.md) | sync your own state across a Photon co-op session | ForgeKit | no |
| [SkillKit](skillkit.md) | ship a custom **learnable skill** (active or passive) | ForgeKit | yes (runtime) |
| [StoryKit](storykit.md) | add an **NPC / trainer** with dialogue and a skill tree | ForgeKit | yes (compile) |
| [EnchantKit](enchantkit.md) | apply vanilla **enchantments** and spawn pre-enchanted gear from a command (dev / testing tool) | ForgeKit | no |
| [DonorKit](donorkit.md) | get a **live creature body** for any species, anywhere in the world | ForgeKit, NetKit | no |
| [CompanionKit](companionkit.md) | give the player a **persistent creature companion** | ForgeKit, AggroKit, NetKit, DonorKit | no |
| [SpawnKit](spawnkit.md) | **spawn creatures** at runtime as real vanilla enemies | ForgeKit, CompanionKit, NetKit, DonorKit | no |

## The layering

Kits stack bottom-up. Everything rests on **ForgeKit**; **CompanionKit** is the mid-tier that
persistent-companion features build on; **SpawnKit** sits on top of it.

```
ForgeKit                      dev tooling — no dependencies
  ├── AggroKit                aggro / threat control
  ├── NetKit                  Photon co-op transport
  ├── SkillKit                custom skills            (+ SideLoader)
  ├── StoryKit                NPCs / trainers          (+ SideLoader)
  ├── EnchantKit              enchant / spawn gear     (dev / testing tool)
  └── DonorKit                creature bodies anywhere (harvest engine + expedition tier + template store; needs ForgeKit + NetKit)
        ├── CompanionKit      persistent companions    (needs ForgeKit + AggroKit + NetKit + DonorKit)
        │     └── SpawnKit    runtime creature spawns  (needs CompanionKit + NetKit + DonorKit)
        └── SpawnKit          also sits directly on DonorKit — its species roster IS the donor table
```

Mods then combine these: [Beastwhispering](../mods/beastwhispering/README.md) uses ForgeKit, SkillKit,
CompanionKit, DonorKit, AggroKit, NetKit, and StoryKit; [Hireling](../mods/hireling.md) uses ForgeKit,
CompanionKit, DonorKit, AggroKit, and NetKit.

## Conventions shared by every kit

- Each ships as its own `BepInEx/plugins/<Kit>/` folder with plugin GUID `cobalt.<name>`.
- Each has a dev-command channel — write a verb into `BepInEx/config/<Kit>_cmd.txt` and it runs on the
  next poll (even while the game is paused). Unknown verb or `help` lists them all. This is
  [ForgeKit](forgekit.md)'s command channel.
- Runtime settings live in `BepInEx/config/cobalt.<name>.cfg`, generated on first launch.

## See also

- [Wiki home](../README.md)
- [Installing](../installing.md)
- [Mods](../mods/README.md) — what these kits are used to build
