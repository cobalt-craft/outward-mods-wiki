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
   the right species is within taming range (**15 m** by default). The chow is consumed and the
   attempt **rolls** — see below. On success the wild creature becomes your pet.

### The roll — taming can fail

Every real attempt has three outcomes:

| attempt | success | it refuses, but **lets you be** | it refuses and **attacks** |
|---|---|---|---|
| normal | 1 in 3 | 1 in 3 | 1 in 3 |
| **from stealth** | 1 in 2 | 1 in 4 | 1 in 4 |

- **From stealth** means you are **sneaking** (crouched) *and* the creature has not noticed you —
  not locked onto you, not already fighting. Sneaking in the middle of a fight does not count.
- **"Lets you be"** — for **60 s** the creature tolerates you: it will not come for you on sight or
  hearing, and if it was already on you it backs off once. The moment **you or your pet hit it**, the
  truce is over and it fights exactly as any wild creature would. Its pack-mates were never party to
  the truce — they will still come.
- **"Attacks"** — it turns on you immediately, and its pack joins.

The chow is consumed on **every roll**, success or failure. The odds are per-install settings today
(see below); per-creature odds are planned.

The tamed creature keeps its own look and its own combat stats — see
[Combat & companion](./combat-and-companion.md).

> **⚠ Known issue — step 3 works in single-player only.** Using the chow currently only tames in a
> **single-player** session; in a co-op session the tame does not take. Everything *after* taming is
> fully multiplayer-ready — an existing pet follows, fights, feeds, bonds and persists in co-op
> exactly as it does solo — so the workaround is to tame solo and then join your session. This is a
> bug and we are working diligently to resolve it.

### No-waste rules

The chow is only consumed when an attempt actually **rolls**. Using it when you **already own a
pet**, or with **no wild target of that species in range**, does nothing — you get a short on-screen
message and keep the chow.

### Settings

In `BepInEx/config/cobalt.beastwhispering.cfg`, section `[Taming]`:

```ini
[Taming]
## Every real taming attempt ROLLS. OFF = every attempt that passes the refusal ladder bonds.
TameRollEnabled = true
## Non-stealth odds (0-1). The attack band is whatever these two leave.
TameSuccessChance = 0.3333333
TamePassiveChance = 0.3333333
## From-stealth odds (sneaking AND unnoticed).
StealthTameSuccessChance = 0.5
StealthTamePassiveChance = 0.25
## How long a "lets you be" creature tolerates you (seconds).
TamePassiveSeconds = 60
```

> BepInEx never migrates a changed default into an existing `.cfg` — an install made before these
> keys existed gets them appended with their defaults on the next launch.

### Which creatures, which chow

Chow matches by species name, so a variant tames with its base species' food — a wild **Armored
Hyena** is tamed with **Hyena Chow**. The one exception is the **Pearlbird Cutthroat**, which cannot
be tamed at all: Pearlbird Chow is refused before it is eaten, so you keep the chow. See
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

## Limitations

- **One pet at a time.** While you have a companion, chow is simply refused ("You already have a
  companion.") — it is never spent and the pet is never swapped. Release the first, or lose it, to
  tame another.
- **Taming itself is single-player only right now** — see the known issue above. The co-op notes
  below describe how the pet systems behave once you bring an already-tamed companion into a
  session.
- **Co-op needs matching versions.** If the host and a guest run different versions of the mod set,
  the session refuses to link the pet systems instead of half-working.

## See also

- [Feeding & diet](./feeding-and-diet.md) — chow ingredients and everyday feeding
- [Loyalty & bonds](./loyalty-and-bonds.md) — keeping (or losing) a pet
- [Creatures](./creatures.md) — the tameable roster
- [Beastwhispering overview](./README.md)
