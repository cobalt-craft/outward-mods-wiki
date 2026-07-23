# Temperature & blankets — keeping your pet comfortable

Your pet feels **heat and cold** just as your character does. Each species is comfortable within its
own temperature range; venturing too far outside it makes the pet suffer — draining its loyalty and,
in the extreme, its health. You keep it comfortable with **blankets** and **weather foods**.

## Comfort bands

The game rates ambient temperature on a nine-step scale from Coldest to Hottest (with Fresh in the
middle). Every species has a **comfort band** — a range of steps it tolerates. The pet's temperature
is sampled at *its* position, so caves, towns, and standing near a fire are all accounted for
automatically.

When the pet is outside its band, discomfort escalates in stages the longer it stays out:

| Stage | How far out of band | Effect |
|---|---|---|
| **Comfortable** | in band | No effect. |
| **Uneasy** | 1 step | Loyalty decays faster (about 1.5×). |
| **Suffering** | 2 steps | Faster decay (about 2×) **and** loses ~2% of max HP per minute. |
| **Critical** | 3+ steps | Faster decay **and** loses ~6% of max HP per minute. |

Escalation isn't instant — the pet has to stay out of band for a sustained stretch to worsen a stage,
and it recovers a stage after getting back into comfort. Both timings are tunable in the
[config reference](./config-reference.md) (`[Systems]`, `[Temperature]`).

### When a pet is drained to zero by the climate

If temperature exposure drives the pet's health to zero, what happens depends on the
`[Temperature] PetDeathMode` setting:

- **Permanent** (default) — the pet dies and the bond is deleted.
- **KnockedOut** — the pet collapses (the normal downed / re-form path).
- **Disabled** — the pet holds at 1 HP, suffering but unkillable by weather.

Climate drain never bypasses this into a normal combat death — it always routes through this setting.

## Relief: blankets

**Blankets** are consumable relief items you craft (Survival-menu recipes) and use on the pet:

- **Heating blanket** — relieves cold.
- **Cooling blanket** — relieves heat.

A blanket wraps the pet for about **15 minutes**, shifting it several steps back toward comfort on that
one side. Wrapping in the same type again refreshes it; the opposite type replaces it. A blanket only
helps against the extreme it's made for — a heating blanket does nothing in the heat, and vice versa.

## Relief: weather foods and water

Fed **weather-resist consumables** — warming and cooling potions, teas, and water — also grant the
pet the hot/cold relief they'd give you, on top of any normal meal:

- Blanket and weather-food relief **stack** on the same side.
- **Every** pet can drink water for cold/heat relief — off its diet, and even when satiated. (Water is
  a drink, not a meal: it grants no loyalty or heal.)

Which consumables count, and how much relief each gives, is data-driven — see
[Feeding & diet](./feeding-and-diet.md) and [Data manifests](./data-manifests.md).

## HUD icons

While a pet is out of its comfort band, cold/freezing or hot/overheating icons appear on your HUD (see
[Loyalty & bonds](./loyalty-and-bonds.md)). The Companion tab shows what the pet is wrapped in and
sipping, and warns if you've wrapped it against the wrong extreme.

## See also

- [Feeding & diet](./feeding-and-diet.md) — weather foods and water
- [Loyalty & bonds](./loyalty-and-bonds.md) — how discomfort speeds loyalty decay
- [Combat & companion](./combat-and-companion.md) — the Companion tab's comfort readout
- [Config reference](./config-reference.md) — `[Temperature]` tuning
- [Beastwhispering overview](./README.md)
