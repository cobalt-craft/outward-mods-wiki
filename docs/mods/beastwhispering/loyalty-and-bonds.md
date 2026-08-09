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
| **Fraying** | 15–54 | Uneasy and weakening. |
| **Steady** | 55–89 | A solid, reliable companion. |
| **Devoted** | 90–100 | Fully bonded. |

The last two floors are deliberately far out (they were 40 and 75 before 2026-08-07): a freshly tamed
pet starts at 50, i.e. **Fraying**, and Steady and Devoted are things you fight your way to.

The tier acts as a "level" (Gone = 0 … Devoted = 4) that scales the pet's stats and the bond buffs it
grants you.

## What changes loyalty

| Event | Change |
|---|---|
| Feed Preferred food | +10 |
| Feed Bond food | +20 |
| Any enemy dies near you while the pet is out | +20 (lands as **+1**) |
| A Hunt as One cast grants Synergy | +20 (lands as **+1**) |
| A For the Kill execute lands the killing blow | +60 (lands as **+3**) |
| First crossing of a region pair (once the Wild Unknown skill is learned) | +5 |
| A day without feeding | −15 (species decay; more in bad weather) |
| The pet is critically hurt | −10 |
| The pet is downed in combat | −20 |

Cold or heat also accelerates the daily decay — see
[Temperature & blankets](./temperature-and-blankets.md).

### Gains are slow — losses are not

A bond is meant to be **hard won**. Every *gain* in that table is scaled down to **5%** of its listed
value before it lands, so a preferred meal is worth **+0.5**, not +10. Every *loss* lands at its full
face value.

**Fighting together is the fast lane.** The three combat rows above carry face values chosen so that,
after that 5% scaling, a kill is worth exactly **one** loyalty and a For-the-Kill killing blow
**three** — which is why they are the only way to climb to Steady and Devoted at any pace. Exactly one
credit is paid per death: an execute kill *upgrades* the plain kill credit instead of stacking on it. Building a Devoted pet is the work of many
sessions of feeding, fighting and travelling together; neglecting one still costs you quickly.

Nothing is lost to rounding. The fraction a gain doesn't deliver is **banked on that pet** and the
next gain adds onto it, so twenty preferred meals hand over exactly the ten loyalty a single
unscaled meal used to. The bank is saved with the pet, so partial progress survives quitting.

The practical consequence: after one meal the loyalty number often **doesn't visibly move**. That is
the system working, not a failed feed — the `petstatus` dev verb reads out how much is banked toward
the next point.

If you'd rather bond at the old pace, set `LoyaltyGainPercent = 100`. Note it scales the combat rows
too, so raising it makes a kill worth twenty loyalty rather than one.

Config lives in `BepInEx/config/cobalt.beastwhispering.cfg`:

```ini
[Systems]
## Percent of every POSITIVE loyalty gain that actually lands.
LoyaltyGainPercent = 5
## Fighting alongside the pet raises the bond (kills, Synergy, For-the-Kill killing blows).
EnableCombatLoyalty = true
## How near you (metres) an enemy must die for the kill to credit the bond. 0 = no distance check.
KillCreditRadius = 40
```

See also the [Config reference](./config-reference.md).

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
