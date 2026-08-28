# Loyalty & bonds — earning (and keeping) your pet's trust

**Loyalty** is a 0–250 value that measures how strongly your pet is bonded to you. It rises when you
care for the pet and falls when you neglect it. Loyalty drives the pet's tier, its bond buffs, the
pet's own power at the highest tiers, and — at the extreme low end — whether it stays with you at
all. A freshly tamed pet starts at **55** — or far higher once you know **The Wild Unknown** (see below).

## Loyalty tiers

Loyalty falls into fourteen tiers. The first eight are the classic bond (0–124); the six above
them are the long tail, where the bond starts making the pet itself stronger.

| Tier | Loyalty | Lock | Meaning |
|---|---|---|---|
| **Gone** | 0 | | The bond is broken — the pet leaves. |
| **Broken** | 1–14 | | Barely holding on. |
| **Fraying** | 15–39 |  | Uneasy and weakening. |
| **Guarded** | 40–54 | | Wary, but the bond is forming. |
| **Cautious** | 55–74 | 🔒 | Wary still, but staying. |
| **Steady** | 75–89 | | A solid, reliable companion. |
| **Trusting** | 90–99 | 🔒 | The pet trusts you deeply. |
| **Devoted** | 100–124 | | The classic bond at its peak. |
| **Unshaken** | 125–149 | 🔒 | Nothing rattles it. The pet grows stronger from here (+1 power step). |
| **Boundless** | 150–174 | | A bond without limit (+2). |
| **Sworn** | 175–199 | 🔒 | An oath neither will break (+3). |
| **Fierce** | 200–224 | | The bond burns fierce (+4). |
| **Mythic** | 225–249 | 🔒 | Sung of in far places (+5). |
| **Eternal** | 250 | 🔒 | The bond at its absolute peak (+6). |

(Tier names were re-spelled on 2026-08-26 — Cautious/Steady/Trusting/Devoted/Unshaken/Boundless/Sworn/Fierce
replaced Steady/Trusting/Devoted/Soulbound/Kindred/Sworn/Fierce/Fabled at the same ordinals. Every
threshold, lock and payout is unchanged; existing saves need nothing.)

A freshly tamed pet starts at 55, i.e. **Cautious**, and the ladder above it is a long climb by design.

### Locks — a bond, once earned, holds

Every tier marked 🔒 is a **lock**: once your pet reaches it, its loyalty can decay down *to* that
tier's floor but never below it. A Trusting pet can never fray back to Broken, and a locked pet can
never be abandoned — neglect still costs you the tiers above the lock, and the pet still goes hungry,
cold and sluggish, but the bond itself is safe. The lock is saved with the pet. Turn it off with
`[Loyalty] EnableLocks = false` if you prefer the old all-or-nothing bond.

### Payouts — eight levels, same ceiling

The bond buffs and other payouts (bond buffs, kill favor, scavenge dice, feed buffs, the execute
heal, the Synergy resistance ceiling) step through **eight payout levels** across the whole ladder.
The old top-of-ladder amounts now sit at **Eternal**; a Trusting or Devoted pet pays **half** of that, and every tier
in between is a step on the way. Several tiers share a level (Guarded/Cautious, Steady through
Devoted, Unshaken/Boundless, Fierce/Mythic), so a new tier name doesn't always mean bigger numbers;
the Companion tab's "next tier" preview always names the tier where the buffs actually grow.

### Power — the pet itself grows

From **Unshaken** up, each tier adds a **power step** to the pet's own stats: +1 to all resistances,
protection, barrier and status resistance, and about +5% max health and damage per step — so an
Eternal pet has +6 of each, ×1.34 health and ×1.37 damage over a Devoted one. This is where the
old species-relic bonuses moved; see the `[Loyalty]` section of the [Config reference](./config-reference.md).

The whole ladder — floors, payouts, locks, power steps — is one table you can retune without a
rebuild (`[Loyalty] LadderOverride`).

### The Wild Unknown head-start

Learning **The Wild Unknown** breakthrough deepens every bond: a pet you tame afterwards starts
**two tiers higher** (at that tier's floor — loyalty **90**, Trusting, with the default start), and
the pet you already have is raised to that same floor **once** when you learn the skill. It's a
one-time gift, not a safety net — if the bond decays below it afterwards, that loss is real.

### The tier strip

The Companion tab (Bond section) draws a row of 14 small boxes just above the *Loyalty* bar — one
per tier, filling green from the left as the bond deepens and greying again if it slips. A tiny padlock marks each lock tier (4, 6, 8, 10, 12 and the top). The padlock is faded and
open until your pet has reached that tier, then solid gold: that floor is secured and decay can never
take the bond below it again. Turn the padlocks off (a plain gap remains) in
`BepInEx/config/cobalt.beastwhispering.cfg`:

```ini
[HUD]
## Draw a tiny padlock at each lock step of the 14-tier loyalty strip in the Companion tab.
ShowLoyaltyLockGlyphs = true
```

## What changes loyalty

| Event | Change |
|---|---|
| Feed Preferred food | +10 |
| Feed Bond food | +20 |
| Any enemy dies near you while the pet is out | +20 (lands as **+1**) |
| A Hunt as One cast grants Synergy | +20 (lands as **+1**) |
| A For the Kill execute lands the killing blow | +60 (lands as **+3**) |
| First crossing of a region pair | +5 |
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
**three** — which is why they are the only way to climb to Cautious and Trusting at any pace. Exactly one
credit is paid per death: an execute kill *upgrades* the plain kill credit instead of stacking on it. Building a Trusting pet is the work of many
sessions of feeding, fighting and travelling together; neglecting one still costs you quickly.

Nothing is lost to rounding. The fraction a gain doesn't deliver is **banked on that pet** and the
next gain adds onto it, so twenty preferred meals hand over exactly the ten loyalty a single
unscaled meal used to. The bank is saved with the pet, so partial progress survives quitting.

The practical consequence: after one meal the loyalty number often **doesn't visibly move**. That is
the system working, not a failed feed — the `petstatus` dev verb reads out how much is banked toward
the next point.

If you'd rather bond at the old pace, set `LoyaltyGainPercent = 100`. Note it scales the combat rows
too, so raising it makes a kill worth twenty loyalty rather than one. At the default rate a bond
feed is worth one point, so each of the six upper tiers is roughly twenty-five bond feeds apart.

Config lives in `BepInEx/config/cobalt.beastwhispering.cfg`:

```ini
[Systems]
## Loyalty a freshly tamed pet starts with (0-250).
InitialLoyalty = 55
## Percent of every POSITIVE loyalty gain that actually lands.
LoyaltyGainPercent = 5
## Fighting alongside the pet raises the bond (kills, Synergy, For-the-Kill killing blows).
EnableCombatLoyalty = true
## How near you (metres) an enemy must die for the kill to credit the bond. 0 = no distance check.
KillCreditRadius = 40

[Loyalty]
## Loyalty LOCKS: a reached lock tier's floor is never decayed below.
EnableLocks = true
## Retune the whole 14-tier ladder: 14 rows of floor[:payout[:lock[:power]]]. Empty = the shipped table.
LadderOverride =
```

See also the [Config reference](./config-reference.md).

### Abandonment

If loyalty falls to **Gone (0)**, the bond breaks and the pet leaves for good — its save is cleared.
Keep a neglected pet fed and comfortable to pull it back from the edge before it reaches zero.

## Bond buffs

A bonded pet passively strengthens **you**. Each species grants a small stat buff scaled by the
loyalty payout level, so an Eternal pet gives the full bonus, a Trusting or Devoted one half of it, and a Broken
one almost nothing. Examples from the shipped roster:

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
