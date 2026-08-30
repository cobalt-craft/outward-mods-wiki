# Creatures — the tameable roster

Not every animal in Outward can be tamed. A creature is **player-tameable** when it has a taming food
(a "chow") to cook and use on it — see [Taming](./taming.md). This page lists the tameable species and
what makes each one distinct.

Beastwhispering ships data for a large catalogue of Outward creatures, but most of that catalogue is
reference and internal support data — the set you can actually tame and keep is the short list below.
Matching is by species name, so a **variant tames with its base species' chow** — a wild Armored Hyena
is tamed with Hyena Chow.

> **Exception: the Pearlbird Cutthroat cannot be tamed.** Pearlbird Chow does not work on one. The
> refusal happens before the chow is eaten, so nothing is consumed — offer it to a base Pearlbird
> instead.

## Tameable species

| Species | Role & signature attack | Bond buff | Notable food / gift |
|---|---|---|---|
| **Hyena** | Brawler. Its bite inflicts **Pain** build-up — stacking hits wear an enemy down. **Evasive:** a hyena keeps its wild reflex — a chance to dodge an incoming melee or ranged strike outright, hopping away from the attacker (10% for a fresh bond, 18% for an Eternal one; never against area blasts). *Ships off in this build — `[Evade] EnableEvade = true` turns it on.* | Physical damage | Eats raw meat, fish, cactus fruit, and Jerky. Chow: pressed meat + fish. Gifts Hide, Raw Salmon, Raw Rainbow Trout, Scaled Leather, and rarely Raw Alpha Meat or Raw Jewel Meat — otherwise Raw Meat. Fed **Metalized Bones**, it gains a permanent stacking bonus. Fed **Alpha Jerky** or **Raw Alpha Meat**, it gets a temporary damage buff scaled by loyalty — and since it eats meat anyway, Raw Alpha Meat feeds it *and* buffs it off the one piece. |
| **Armored Hyena** | Tank. **Brace** — Hunt as One opens a counterattack stance that negates incoming blows and ripostes with heavy impact and a random debuff (Weaken or Sapped). Entering the stance also **pins its current target's aggro onto the pet**, for longer the deeper the bond. Tamed with **Hyena Chow**. | Physical damage | Same diet, gift table, **Metalized Bones** relic, and **Alpha Jerky** / **Raw Alpha Meat** damage buffs as the Hyena. |
| **Pearlbird** | Support striker. Its signature attack inflicts **Cripple**. | Movement speed | Eats gaberries, vegetables, and any mushroom (mushrooms give a little less than vegetables). Chow: maize + gaberries. Gifts eggs — and, taught the gift skill, quality feathers for fletching arrows. Fed **Pearlbird's Courage**, it gains a permanent stacking bonus. |
| **Veaber** | Hex-dealer. Its signature attack inflicts **hexes chosen by what you've recently fed it** — a scavenger whose diet shapes its bite. | All elemental damage (Ethereal, Decay, Electric, Frost, Fire) | Scavenger diet (food waste, seeds, oils, bones). Chow: food waste + crystal powder + mushroom. Gifts eggs. |

Each species also has its own comfort band, gift table, and — for some — scent tracking, scavenge
bonuses, and relic feeds. Those details, and the full per-creature data, live in
[Data manifests](./data-manifests.md).

## Signature attacks at a glance

The shipped species deliberately play differently through their signature (Hunt as One) attacks:

- **Status bite** (Hyena) — build up a debuff over repeated hits.
- **Counterattack stance** (Armored Hyena) — a defensive tank that turns enemy blows into ripostes, and
  taunts its target into swinging at the pet rather than at you.
- **Debuff striker** (Pearlbird) — reliable crowd control.
- **Adaptive hexes** (Veaber) — its effect depends on its recent meals.

Some species are defined in the data with signature attacks that aren't reachable through the player
taming loop yet — for example the **Mantis Shrimp**, whose signature fires its own ranged electric
bolt. As the roster grows, more of these become tameable; see [Data manifests](./data-manifests.md)
for how species data is authored and extended.

## See also

- [Taming](./taming.md) — how to tame each of these
- [Feeding & diet](./feeding-and-diet.md) — full diets
- [Combat & companion](./combat-and-companion.md) — how signature attacks fit into combat
- [Data manifests](./data-manifests.md) — the complete creature dataset
- [Beastwhispering overview](./README.md)
