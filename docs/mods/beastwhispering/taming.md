# Taming — turning a wild animal into your pet

**Taming** is how you gain a pet in Beastwhispering. You cook a species-specific food and use it next
to a wild animal of that species; the wild creature becomes your companion. You keep **one pet at a
time**.

## The taming loop

1. **Get the recipe.** Once you've learned the **Scatology** pet skill (from the pet trainer — see
   [Skills](./skills.md)), every tameable creature drops a **recipe scroll** for its taming food when
   it dies. Read the scroll to learn the recipe. (With Scatology learned the drop is guaranteed by
   default; both the skill requirement and the drop rate are configurable.)
2. **Cook the taming food.** Each tameable species has its own food — its **"chow"** — cooked at a
   **cooking pot** from a handful of ingredients. One craft yields a few chow.
3. **Use the chow near a wild one.** With the chow in your inventory, use it while a wild creature of
   the right species is within taming range (**15 m** by default). The chow is consumed and the wild
   creature becomes your pet.

The tamed creature keeps its own look and its own combat stats — see
[Combat & companion](./combat-and-companion.md).

### No-waste rules

The chow is only consumed on a successful tame. Using it when you **already own a pet**, or with **no
wild target of that species in range**, does nothing — you get a short on-screen message and keep the
chow.

### Which creatures, which chow

Chow matches by species name, so a variant tames with its base species' food — a wild **Armored
Hyena** is tamed with **Hyena Chow**, and a **Pearlbird Cutthroat** with **Pearlbird Chow**. See
[Creatures](./creatures.md) for the tameable roster and [Feeding & diet](./feeding-and-diet.md) for
what each chow is made of.

> **Note on cooking:** Outward's freeform cooking matches recipes you haven't formally learned and
> auto-learns them on a successful cook. The recipe scroll is the intended discovery path, not a hard
> gate — a lucky cook can teach the recipe too.

## Recall

Your pet follows you automatically across zones, doors, resting, and reloads. If it ever gets stuck,
misplaced, or fails to reappear, press **F9** (Recall). Recall warps the pet to your feet and re-forms
it if its body was lost. It is the in-game remedy for any wandering or missing pet.

## Release

Releasing a pet frees it and clears its saved bond — you can then tame a new one. Release is available
through the `release` dev command (see [Dev verbs](./dev-verbs.md)). Abandonment (letting loyalty fall
to nothing) also ends the bond permanently — see [Loyalty & bonds](./loyalty-and-bonds.md).

## See also

- [Feeding & diet](./feeding-and-diet.md) — chow ingredients and everyday feeding
- [Loyalty & bonds](./loyalty-and-bonds.md) — keeping (or losing) a pet
- [Creatures](./creatures.md) — the tameable roster
- [Beastwhispering overview](./README.md)
