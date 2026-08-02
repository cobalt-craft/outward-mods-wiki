# Beastwhispering — tame wild Outward animals as pets

**Beastwhispering** is a mod for Outward that lets you tame wild animals into persistent pets. A tamed
creature follows you across zones and reloads, feeds and bonds with you, keeps its own species stats,
fights at your side, and grows more useful the more it trusts you.

**At a glance**
- Type: gameplay mod
- Requires: BepInEx 5 (Mono branch), SideLoader, and its kits ([ForgeKit](../../kits/forgekit.md),
  [SkillKit](../../kits/skillkit.md), [CompanionKit](../../kits/companionkit.md),
  [StoryKit](../../kits/storykit.md), [AggroKit](../../kits/aggrokit.md),
  [NetKit](../../kits/netkit.md)) — all pulled in automatically. See [Installing](../../installing.md).
- Config: `BepInEx/config/cobalt.beastwhispering.cfg`
- Commands: `BepInEx/config/bw_cmd.txt`

## For players

You tame an animal, and it becomes a companion that stays with you — through doors, zone changes,
resting, and even quitting and reloading the game. From there it is a small creature to look after:
it gets hungry, it feels the weather, it grows loyal (or drifts away if neglected), and it fights
what you fight. A native **Companion** tab in the character menu shows its full sheet at a glance.

Only certain species can be tamed — see [Creatures](./creatures.md).

> **⚠ Known issue — taming is single-player only.** Using chow on a wild creature currently only
> succeeds in a **single-player** session. An already-tamed pet is **fully playable in multiplayer**:
> it follows, fights, feeds, bonds and persists in co-op just as it does solo. Tame your companion
> solo, then take it into co-op. This is a bug, not a design decision, and we are working diligently
> to resolve it.

### Getting started

The core loop:

1. **Find a tameable animal** (a wild Hyena is the easiest first target).
2. **Get its taming-food recipe.** Once you've learned the **Scatology** pet skill (bought from the
   pet trainer — see [Skills](./skills.md)), tameable creatures drop a recipe scroll when they die.
   Read the scroll to learn the recipe. (You can also stumble onto a recipe through Outward's freeform
   cooking — see [Taming](./taming.md).)
3. **Cook the taming food** ("chow") at a cooking pot from the recipe's ingredients.
4. **Use the chow near a wild one** of that species. The wild creature becomes your pet.
5. **Feed and bond.** Feeding restores its health and raises loyalty; a hungry, cold, or neglected pet
   loses loyalty and can eventually leave.
6. **Fight together.** Your pet auto-assists what you attack, and you can teach it signature skills.

You keep **one pet at a time**. Full details in [Taming](./taming.md).

### Default keybinds

| Key | Action |
|---|---|
| **F7** | Tame the nearest wild creature |
| **F8** | Feed the pet the first inventory item its diet accepts |
| **F9** | Recall the pet to your feet (also re-forms a missing pet — the fix for a stuck pet) |
| **F10** | Run the self-test |
| **F12** | Log a diagnostic snapshot |

All keys are rebindable in `cobalt.beastwhispering.cfg` (`[Keys]`). Keybinds are shared across every
mod in this family, so a clash is reported at startup — see the
[config reference](./config-reference.md).

## Pages

| Page | What it covers |
|---|---|
| [Taming](./taming.md) | Recipe scrolls, cooking chow, taming a wild animal, recall and release |
| [Feeding & diet](./feeding-and-diet.md) | What each species eats, hunger, healing, special food kinds |
| [Loyalty & bonds](./loyalty-and-bonds.md) | Loyalty tiers, what raises and lowers them, bond buffs, HUD icons |
| [Temperature & blankets](./temperature-and-blankets.md) | Comfort bands, cold/heat effects, blankets and weather foods |
| [Combat & companion](./combat-and-companion.md) | How your pet fights, the Companion tab, species-true stats |
| [Creatures](./creatures.md) | Which creatures are tameable and their signature traits |
| [Skills](./skills.md) | The learnable pet skills (Hunt as One, Command Pet, Communion, and more) |
| [Config reference](./config-reference.md) | Every setting in `cobalt.beastwhispering.cfg` |
| [Dev verbs](./dev-verbs.md) | The `bw_cmd.txt` command channel for testing |
| [Data manifests](./data-manifests.md) | The per-species data files and how to add creatures |

## See also

- [Mods index](../README.md)
- [Installing](../../installing.md)
- [CompanionKit](../../kits/companionkit.md) — the companion spine Beastwhispering is built on
- [Wiki home](../../README.md)
