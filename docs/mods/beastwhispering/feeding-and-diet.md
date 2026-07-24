# Feeding & diet — keeping your pet fed

**Feeding** is the heart of the bond. A fed pet heals, gains loyalty, and slowly regenerates health
over time; a pet left unfed grows hungry and drifts away. Each species eats different things.

## How feeding works

Offer the pet food two ways:

- **F8** — feed the first item in your inventory that the pet's diet accepts.
- The right-click **Feed** action on a food item.

A successful feed:

- **Restores health** immediately (a small heal, per food).
- **Raises loyalty** — Preferred food and the stronger **Bond food** each grant a fixed amount (see
  [Loyalty & bonds](./loyalty-and-bonds.md)).
- **Starts an over-time health regen** — a vanilla-style Health Recovery buff that heals the pet
  gradually for several minutes. A new feed replaces any running regen. A species' own taming chow
  always grants the strongest regen.

### Diets: Preferred vs Bond food

Every species has its own diet list. Foods come in two kinds:

| Kind | Loyalty | Notes |
|---|---|---|
| **Preferred** | +10 (default) | Ordinary foods the species likes. |
| **Bond** | +20 (default) | The species' favourite — usually its own chow. Outranks Preferred when both match. |

A diet entry can be a specific item or a whole food category (any Meat, any Fish, any Vegetable, and
so on). The exact per-species diets live in the data files — see [Data manifests](./data-manifests.md).

### Hunger and satiation

Pets get hungry on a timer. If a pet goes a full "day" without eating, it loses loyalty to hunger,
and a hungry pet shows a status icon on your HUD. A **recently fed** pet is satiated and refuses all
food — even its Bond food — until it has digested. This means you can't spam-feed to farm loyalty; you
feed on a rhythm.

Hunger timing is tunable, and can differ per species. See the
[config reference](./config-reference.md) (`[Systems]`) and [Data manifests](./data-manifests.md).

## Special food kinds

Beyond ordinary diet food, several consumables do more than fill the belly:

| Food kind | What it does |
|---|---|
| **Taming chow** | The food used to tame the species. As a Bond food it also feeds and grants the strongest health regen. See [Taming](./taming.md). |
| **Weather foods & drinks** | Warming/cooling potions, teas, and water give the pet temperature relief when fed, on top of any normal meal. **Every** pet can drink water (off-diet and even when satiated). See [Temperature & blankets](./temperature-and-blankets.md). |
| **Buff foods** | Certain species turn specific reagents into a temporary combat buff (for example, the Veaber and mineral dust, or a Hyena/Armored Hyena fed Alpha Jerky or Raw Alpha Meat — a physical damage buff scaled by loyalty). |
| **Relics** | A few species accept a specific vanilla relic for a permanent, stacking bonus — feeding a base Pearlbird its Courage relic, or a base Veaber a Leyline Figment. These are not meals; they don't fill hunger or heal. |

Which foods, buffs, and relics a species takes is data-driven per creature — see
[Data manifests](./data-manifests.md).

## See also

- [Loyalty & bonds](./loyalty-and-bonds.md) — how feeding raises the bond
- [Temperature & blankets](./temperature-and-blankets.md) — weather foods and water
- [Creatures](./creatures.md) — per-species notable foods
- [Data manifests](./data-manifests.md) — the full diet data
- [Beastwhispering overview](./README.md)
