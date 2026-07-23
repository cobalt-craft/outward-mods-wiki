# Creatures — the tameable roster

Not every animal in Outward can be tamed. A creature is **player-tameable** when it has a taming food
(a "chow") to cook and use on it — see [Taming](./taming.md). This page lists the tameable species and
what makes each one distinct.

Beastwhispering ships data for a large catalogue of Outward creatures, but most of that catalogue is
reference and internal support data — the set you can actually tame and keep is the short list below.
Matching is by species name, so a **variant tames with its base species' chow** (a wild Armored Hyena
with Hyena Chow, a Pearlbird Cutthroat with Pearlbird Chow).

## Tameable species

| Species | Role & signature attack | Bond buff | Notable food / gift |
|---|---|---|---|
| **Hyena** | Brawler. Its bite inflicts **Pain** build-up — stacking hits wear an enemy down. | Physical damage | Eats raw meat, fish, cactus fruit. Chow: pressed meat + fish. Gifts raw meat and scaled leather. |
| **Armored Hyena** | Tank. **Brace** — Hunt as One opens a counterattack stance that negates incoming blows and ripostes with heavy impact and a random debuff (Weaken or Sapped). Tamed with **Hyena Chow**. | Physical damage | Same diet as the Hyena. |
| **Pearlbird** | Support striker. Its signature attack inflicts **Cripple**. | Movement speed | Eats gaberries and vegetables. Chow: maize + gaberries. Gifts eggs — and, taught the gift skill, quality feathers for fletching arrows. |
| **Veaber** | Hex-dealer. Its signature attack inflicts **hexes chosen by what you've recently fed it** — a scavenger whose diet shapes its bite. | Decay damage | Scavenger diet (food waste, seeds, oils, bones). Chow: food waste + crystal powder + mushroom. Gifts eggs. |

Each species also has its own comfort band, gift table, and — for some — scent tracking, scavenge
bonuses, and relic feeds. Those details, and the full per-creature data, live in
[Data manifests](./data-manifests.md).

## Signature attacks at a glance

The shipped species deliberately play differently through their signature (Hunt as One) attacks:

- **Status bite** (Hyena) — build up a debuff over repeated hits.
- **Counterattack stance** (Armored Hyena) — a defensive tank that turns enemy blows into ripostes.
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
