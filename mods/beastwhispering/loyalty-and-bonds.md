# Loyalty & bonds — earning (and keeping) your pet's trust

**Loyalty** is a 0–100 value that measures how strongly your pet is bonded to you. It rises when you
care for the pet and falls when you neglect it. Loyalty drives the pet's tier, its bond buffs, and — at
the extreme low end — whether it stays with you at all. A freshly tamed pet starts at **50**.

## Loyalty tiers

Loyalty falls into five tiers:

| Tier | Loyalty | Meaning |
|---|---|---|
| **Gone** | 0 | The bond is broken — the pet leaves. |
| **Broken** | 1–14 | Barely holding on. |
| **Fraying** | 15–39 | Uneasy and weakening. |
| **Steady** | 40–74 | A solid, reliable companion. |
| **Devoted** | 75–100 | Fully bonded. |

The tier acts as a "level" (Gone = 0 … Devoted = 4) that scales the pet's stats and the bond buffs it
grants you.

## What changes loyalty

| Event | Change |
|---|---|
| Feed Preferred food | +10 |
| Feed Bond food | +20 |
| Defeat a nearby difficult enemy together | +5 |
| First crossing of a region pair (once the Wild Unknown skill is learned) | +5 |
| A day without feeding | −15 (species decay; more in bad weather) |
| The pet is critically hurt | −10 |
| The pet is downed in combat | −20 |
| You flee combat, leaving the pet engaged | −15 |

Cold or heat also accelerates the daily decay — see
[Temperature & blankets](./temperature-and-blankets.md).

### Abandonment

If loyalty falls to **Gone (0)**, the bond breaks and the pet leaves for good — its save is cleared.
Keep a neglected pet fed and comfortable to pull it back from the edge before it reaches zero.

## Bond buffs

A bonded pet passively strengthens **you**. Each species grants a small stat buff scaled by the
loyalty tier, so a Devoted pet gives the full bonus and a Broken one gives almost nothing. Examples
from the shipped roster:

| Species | Bond buff (at full loyalty) |
|---|---|
| Hyena | Physical damage |
| Pearlbird | Movement speed |
| Veaber | Decay damage |

Some species also grant extra backpack capacity as part of the bond. The exact buffs are data-driven
per creature — see [Data manifests](./data-manifests.md).

> **Bond buffs unlock through a skill.** By default the passive bond buffs (and the bag capacity gift)
> are gated behind the **Communion** pet skill — you learn it to start receiving them, and the
> Companion tab shows what the bond is granting. See [Skills](./skills.md).

## HUD status icons

Your pet's state shows as indicator-only status icons on your own HUD, so you can read it at a glance
without opening a menu:

- **Hunger** — a Hungry icon appears as the pet approaches a full hunger day, a Starving icon at a
  full day.
- **Bond** — icons mark a fraying or broken bond, and a steady or devoted one.
- **Temperature** — cold/freezing and hot/overheating icons (see
  [Temperature & blankets](./temperature-and-blankets.md)).

A well-fed, comfortable, well-bonded pet shows nothing — the icons are warnings and highlights, not
clutter.

## See also

- [Feeding & diet](./feeding-and-diet.md) — the main way to raise loyalty
- [Temperature & blankets](./temperature-and-blankets.md) — comfort and its effect on loyalty
- [Skills](./skills.md) — Communion and the other pet skills
- [Combat & companion](./combat-and-companion.md) — the Companion tab reads out loyalty live
- [Beastwhispering overview](./README.md)
