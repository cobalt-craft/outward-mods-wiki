# Beastwhispering skills — the beastwhisperer's tree

**Beastwhispering** adds ten learnable skills, sold as a single skill tree by a trainer NPC. They turn
a tamed animal from a pet that merely follows you into a genuine combat partner — and gate some of the
mod's deeper systems behind real progression.

For how taming itself works, see [Taming](./taming.md); for how the pet fights, see
[Combat & companion](./combat-and-companion.md).

## Where to learn them

The skills are taught by **Maren the Beastwhisperer**, a trainer who stands in **Cierzo** (near the
blacksmith). She is a normal Outward trainer — talk to her, choose **"Train with me."**, and the
standard trainer window opens. That window handles everything the way any vanilla trainer does:

- **Silver cost** per skill (shown in the table below).
- **Prerequisites** — a skill greys out until you've learned the one it branches from.
- **The breakthrough** — the tree has one breakthrough skill (Wild Unknown). As with vanilla trees,
  you may learn only **one** breakthrough per tree, and the top rows unlock only after you take it.

Maren and her dialogue are built on **[StoryKit](../../kits/storykit.md)**, the NPC/trainer library.
If she isn't where you expect, the mod can re-place her (see the `marenhere` verb in
[Dev verbs](./dev-verbs.md)); she can also be disabled entirely from the config.

## The tree

The tree is called **Beastwhispering**. It has a passive spine up the middle — Scatology → the
row-2 branch → **Wild Unknown** (the breakthrough) → the row-4 branch — with the active combat skills
hanging off the outer columns. Rows are listed bottom (entry) to top.

| Skill | Type | Cost | Prerequisite | Effect |
|---|---|---:|---|---|
| **Scatology** | Passive | 50 | — (entry) | Slain tameable creatures may drop their **taming-food recipe scroll**. This is the discovery path for taming any species. |
| **Command Pet** | Active | 100 | Scatology | One button, three orders in a cycle: Engage → Follow → Stay. **Engage**: send the pet at your locked target, or the nearest enemy in front of you. **Follow**: recall it to your side to wait passively. **Stay**: it holds the spot it is standing on — until your next order, or until you get far enough away that it has to come find you (then it is following again). **Stay is fully passive: a staying pet will not defend itself.** If something attacks it while it holds the spot, it will not fight back, and it can be killed there. Use Stay to park a pet somewhere safe, not to post a guard. |
| **Beast of Burden** | Passive | 100 | Scatology | +10 carrying capacity while a pet travels with you, split between the two of you: **+7 on the pet's own inventory** and **+3 on your pouch**. Your backpack is not affected. |
| **Release Pet** | Active | 50 | Scatology | End the bond for good — the animal returns to the wild and the pet is gone. |
| **Wild Unknown** | Passive · **breakthrough** | 500 | (breakthrough) | Your pet **fights at full strength** (without it, its damage is throttled — 50% by default), and every beast you tame afterwards starts **two loyalty tiers higher**. As the tree's breakthrough it also opens Hunt as One, Communion and Heal Pet. It does **not** affect region travel or the first-crossing loyalty bonus — you get both without it. |
| **Hunt as One** | Active | 500 | Wild Unknown | Command the pet to unleash its **signature attack**; you strike alongside it on the same target for bonus damage. Landing both halves builds **Synergy** (a stacking damage buff). Some species also **taunt** with it — the Armored Hyena's Brace stance pins its target's aggro onto the pet, for longer the deeper the bond. |
| **Communion** | Passive | 600 | Wild Unknown | Attunes you to the pet: unlocks the **bond's passive gifts** (per-species stat buffs and the bag-capacity bonus), shown as a tier-coloured badge you can read at a glance. |
| **Heal Pet** | Active | 500 | Wild Unknown | Channel your bond to restore the pet to full health. |
| **For the Kill** | Active | 600 | Hunt as One | Spend **every stack of Synergy** in one coordinated execute — you and the pet strike together, hitting harder for each stack consumed, and the pet's species leaves its own debuff on the prey. |
| **Gift of the Wild** | Active | 600 | Heal Pet | Ask the companion for a gift. What it brings — if anything — depends on its species and how deep your bond runs. (Pearlbirds gift feathers you can fletch onto arrows.) |

Costs are the silver prices Maren charges and may be tuned in the config.

## Skills that gate other systems

Four of the passives are not just abilities — they switch parts of the mod on:

- **Scatology → recipe-scroll drops.** Until someone in your session has learned it, tameable
  creatures don't drop the recipe scrolls you cook their taming food from. (Any player character in a
  co-op session counts, not only the host.)
- **Wild Unknown → full pet damage.** Without the breakthrough your pet's damage is throttled; with
  it, the pet hits at full strength (and new tames start much more loyal). It does **not** gate
  travel or the first-crossing loyalty bonus: **your pet follows you across every region and earns
  that bonus either way**, skill or not.
- **Beast of Burden → carry capacity.** The capacity follows the *bond*, not the body — a downed pet
  still carries for you. It is split **70/30**: 70% onto the pet (on top of whatever its species can
  already carry) and 30% onto your pouch. The total is one setting,
  `[Skills] BeastOfBurdenCapacity` in `BepInEx/config/cobalt.beastwhispering.cfg` (default `10`, so
  +7/+3); `0` switches it off. This skill adds nothing to your equipped backpack — the backpack
  bonus a Phytosaur grants is a separate per-species gift.
- **Communion → bond buffs.** The pet's passive player buffs (stat bonuses from its species, plus the
  bag-capacity gift) only apply once Communion is learned. Before that, a pet grants no passive buff.

Each of these gates can be turned off in the config (see [Config reference](./config-reference.md),
the `[Skills]` section) to restore the older "always on" behaviour.

## Using the active skills

The active skills (Hunt as One, For the Kill, Heal Pet, Gift of the Wild, Command Pet, Release Pet)
work like any Outward skill: assign one to a quickslot and press it. Command Pet's quickslot icon
changes to name the action your **next** press will perform (Follow, Stay or Engage). Hunt as One and For
the Kill play a real weapon flourish, so they work with a melee weapon or a bow equipped.

## See also

- [Beastwhispering overview](./README.md)
- [Combat & companion](./combat-and-companion.md) — Synergy, signature attacks, the anchor
- [Loyalty & bonds](./loyalty-and-bonds.md) — tiers, decay, and the passive buffs Communion unlocks
- [Taming](./taming.md) — recipe scrolls, chow, and the taming loop
- [Config reference](./config-reference.md) · [Dev verbs](./dev-verbs.md)
- [StoryKit](../../kits/storykit.md) — the trainer/dialogue library Maren is built on
- [Mods index](../README.md)
