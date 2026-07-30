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

> **Those are face values.** Loyalty gains land at **5%** of them by default (a preferred meal is
> worth +0.5), with the remainder banked on the pet so it still adds up over many meals — bonding is
> meant to be slow. See [Gains are slow — losses are not](./loyalty-and-bonds.md#gains-are-slow--losses-are-not).

A diet entry can be a specific item or a whole food category (any Meat, any Fish, any Vegetable, and
so on) — a Pearlbird, for instance, takes **any mushroom**, for a slightly smaller loyalty and health
gain than a vegetable. The exact per-species diets live in the data files — see
[Data manifests](./data-manifests.md).

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
| **Weather foods & drinks** | Warming/cooling potions, teas, and water give the pet temperature relief when fed, on top of any normal meal. **Every** pet can drink water (off-diet and even when satiated). A fed **Weather Defense Potion** goes further: total immunity to weather for 10 minutes, and every pet drinks it. See [Temperature & blankets](./temperature-and-blankets.md). |
| **Buff foods** | Certain species turn specific consumables into a temporary combat buff (for example, the Veaber and mineral dust, or a Hyena/Armored Hyena fed **Alpha Jerky** or **Raw Alpha Meat** — a temporary damage buff scaled by loyalty). Whether a buff food *also* fills the belly depends on the species' diet, and the two are decided independently: a buff food the diet doesn't accept (mineral dust, Alpha Jerky) is buff-only — no hunger, no heal, and never picked by the auto-feed key. A buff food the diet *does* accept (**Raw Alpha Meat**, which both hyenas eat as meat) is fed as a normal meal **and** grants the buff, for a single item. |
| **Relics** | Some species accept one specific vanilla relic for a permanent, stacking bonus, **up to 5 stacks** — after that the relic is refused and not consumed. These are not meals; they don't fill hunger or heal. |

Relics are matched per species, so each species (including variants) takes exactly one relic:

| Species | Relic |
|---|---|
| Pearlbird | Pearlbird's Courage |
| Veaber | Leyline Figment |
| Hyena | Metalized Bones |
| Armored Hyena | Metalized Bones |

Offering a relic to a species it doesn't belong to is simply "not interested" — the pet then judges the
item against its ordinary diet instead.

Which foods, buffs, and relics a species takes is data-driven per creature — see
[Data manifests](./data-manifests.md).

## See also

- [Loyalty & bonds](./loyalty-and-bonds.md) — how feeding raises the bond
- [Temperature & blankets](./temperature-and-blankets.md) — weather foods and water
- [Creatures](./creatures.md) — per-species notable foods
- [Data manifests](./data-manifests.md) — the full diet data
- [Beastwhispering overview](./README.md)
