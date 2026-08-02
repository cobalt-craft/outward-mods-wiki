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
| **Suffering** | 2 steps | Loyalty decays faster (about 2×) **and** the pet loses ~2% of max HP per minute. |
| **Critical** | 3+ steps | Same ~2× loyalty decay as Suffering, but the pet loses ~6% of max HP per minute. |

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

### Finding the recipes

Each blanket also has a **recipe scroll** that can turn up in **world loot containers** — chests
(Chest, Ornate Chest, Trog Chest, Stash, Supply Cache) and the odds-and-ends containers (Broken
Tent, Hollowed Trunk, Junk Pile). The two scrolls are rolled **independently**, at **5% each** per
container (`[Temperature] BlanketRecipeDropChance` in `BepInEx/config/cobalt.beastwhispering.cfg`);
set it to `0` if you would rather the recipes stayed craft-only.

```ini
[Temperature]
## Chance (0-1) that each blanket recipe scroll drops from a world loot container.
BlanketRecipeDropChance = 0.05
```

They do **not** drop from creatures you kill, and not from the corpse containers you find lying
around either — chests and the odd-container set only. A container rolls once when it fills its
contents, so re-opening the same chest never yields more; when an area resets (about seven in-game
days) its chests re-roll, ours along with the vanilla loot.

## Relief: weather foods and water

Fed **weather-resist consumables** — warming and cooling potions, teas, and water — also grant the
pet the hot/cold relief they'd give you, on top of any normal meal:

- Blanket and weather-food relief **stack** on the same side.
- **Every** pet can drink water for cold/heat relief — off its diet, and even when satiated. (Water is
  a drink, not a meal: it grants no loyalty or heal.)

### Total immunity: the Weather Defense Potion

A fed **[Weather Defense Potion](https://outward.fandom.com/wiki/Weather_Defense_Potion)** is the one
absolute: for its **10 minutes** the pet is **completely immune to weather** — no cold, no heat, no
matter how far outside its comfort band it is. This is not step relief that a harsh enough climate can
outrun; while it runs the pet simply cannot be made uncomfortable by temperature, so there is no
loyalty decay and no health drain from the climate.

Like water, the potion is drinkable by **every** pet — off its diet and even when satiated (no species
eats potions, so it would otherwise be un-feedable). It's a drink, not a meal: no loyalty, no heal, no
change to hunger. The Companion tab's *Sipped* row reads "immune to the weather" while it lasts.

Which consumables count, and how much relief each gives, is data-driven — see
[Feeding & diet](./feeding-and-diet.md) and [Data manifests](./data-manifests.md).

## HUD icons

While a pet is out of its comfort band, cold/freezing or hot/overheating icons appear on your HUD (see
[Loyalty & bonds](./loyalty-and-bonds.md)). The Companion tab shows what the pet is wrapped in and
sipping, and warns if you've wrapped it against the wrong extreme.

## Limitations

- **The recipe-scroll drop is rare by design** (5% per container, per scroll). If you'd rather not
  rely on it, both blankets stay craftable from their ingredients once you know the recipe.
- **Relief is one-sided and can be outrun.** In the harshest zones a single blanket may not fully
  cover a very cold- or heat-sensitive species; stack a weather food or drink on top, or leave.
- **A pet drained to zero by the climate is gone for good under the default `PetDeathMode`.** If you
  would rather that never happen, set it to `KnockedOut` or `Disabled` before travelling somewhere
  extreme.

## See also

- [Feeding & diet](./feeding-and-diet.md) — weather foods and water
- [Loyalty & bonds](./loyalty-and-bonds.md) — how discomfort speeds loyalty decay
- [Combat & companion](./combat-and-companion.md) — the Companion tab's comfort readout
- [Config reference](./config-reference.md) — `[Temperature]` tuning
- [Beastwhispering overview](./README.md)
