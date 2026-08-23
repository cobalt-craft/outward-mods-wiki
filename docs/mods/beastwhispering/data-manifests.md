# Beastwhispering data manifests — the creature data model

Everything Beastwhispering knows about a creature — what it eats, what climate suits it, its signature
attack, what it gifts, the taming food you cook to catch it — lives in **one JSON file per creature**
under `data/species/`. There are around **167** of them (every creature in the game gets a file, even
the ones that aren't tameable yet). A command-line tool, **`bwspecies`**, compiles those manifests into
the compact tables the mod ships embedded in its DLL.

This page is for modders who want to add a creature, change what an existing one does, or understand
where the mod's per-species behaviour comes from. Players don't touch any of this.

## One file per creature

Each `data/species/<Name>.json` is a JSON object (with `//` line comments allowed) holding a set of
optional **axes**. A creature only carries the axes it actually uses — the `Default.json` file supplies
fallbacks, and most creatures define only a few axes. For example, `Hyena.json` carries a comfort band,
a diet, a taming food, gifts, senses, a signature attack, a passive buff, a growth curve and a
for-the-kill debuff; a plain undead might carry only a comfort band and donor scenes.

The two identity fields every file has:

| Field | Meaning |
|---|---|
| `species` | The creature's display name (matched against the live creature at runtime). |
| `order` | Sort order in listings / tables. |

Free-text `notes` may sit alongside; they are documentation only and are never compiled into a table.

## The axes

| Axis | Governs | Ships as |
|---|---|---|
| `diet` | Foods the pet accepts, with loyalty/heal values (per item or per food **category**) | `PetDiets.json` |
| `tamingFood` | The chow + recipe scroll you cook to tame it, and the recipe's ingredients | `TamingFoods.json` |
| `comfort` | Temperature comfort band (`min`/`max` steps) | `SpeciesComfort.txt` |
| `buffs` | Passive player buffs the bond grants (stat + percent per loyalty level) | `SpeciesBuffs.txt` |
| `signatureAttack` | The Hunt as One special: trigger, windup, status, damage mult, cooldown, build-up, `Kind` (Melee / Ranged / Brace), and an optional `taunt` — `minSeconds`/`maxSeconds` (the aggro-pin window, lerped over loyalty) plus `chancePercent`/`chancePerStack` for a pin earned through Hunt-as-One **Synergy** (the roll at one stack, and how much each further stack adds). Both chances absent = the pin is stance-earned only; the whole window absent = no taunt at all | `SpeciesSpecialAttacks.txt` |
| `forTheKill` | The For the Kill execute debuff, plus an optional `killBuff` | `ForTheKill.json` |
| `foodHexes` | Which fed foods add which hex build-up on Hunt as One | `FoodHexes.json` |
| `gifts` | The Gift skill's drop table (default drop + loyalty-lerped drops + nothing-chance) | `PetGifts.json` |
| `senses` | Interesting item/creature spawns the pet noses out ("scent") | `PetSenses.json` |
| `scavenge` | Loot containers the pet rolls extra times, and how many per loyalty tier | `PetScavenge.json` |
| `sigils` | How the Hunt as One hit changes while the pet stands in a mage sigil | `Sigils.json` |
| `buffFoods` | Fed consumables that grant a temporary damage buff. Independent of `diet`: an item in both is fed as a meal **and** buffs, off one item; an item only here buffs without feeding | `BuffFoods.json` |
| `skillEchoes` | Per-skill overrides for the pet's bonus strike on your weapon skills | `SkillEchoes.json` |
| `loyaltyGrowth` | How the pet's stats grow with loyalty (per stat group, percent at 100) | `SpeciesGrowth.txt` |
| `flourish` | The animation the creature plays when a spell you cast reaches it. Either a plain trigger string (`"Flourish"`) or `{ "default": …, "perSpell": { "ward": … } }` for a creature whose animation differs by spell. **Empty by default — a creature with no entry animates nothing**, which is the correct starting state: trigger names are per-creature facts that can only be read off a live body with the `animdump` dev command, never guessed. (The *visual* half of a spell belongs to the spell, not the creature, and lives in `PetSpellFx.json`.) | `SpeciesFlourish.txt` |
| `donorScenes` / `donorObject` | Which scenes the creature's body can be harvested from | `DonorScenes.txt` ([DonorKit](../../kits/donorkit.md)) |
| `yaw` | Rig-facing correction if the model walks sideways/backward | `SpeciesYawOffsets.txt` |
| `carry` | `{ "weight": … }` — how much item weight this species' pet inventory holds. No entry = the `[Systems] PetCarryWeightDefault` backfill | `SpeciesCarry.txt` |
| `slopeTilt` | `true`/`false` — does the body pitch and roll to match the ground? An anatomy fact: quadrupeds tilt, bipeds stay upright. No entry = upright | `SpeciesSlopeTilt.txt` |
| `combatStyle` | `"chase"` or `"opposite"` — does the pet chase its target, or station **across** the enemy from you so your arrow corridor stays open? No entry = chase | `SpeciesCombatStyle.txt` |
| `synergyResist` | `{ "multiplier": … }` — this species' term in the all-resistances bonus its body earns while Synergy stacks are up (loyalty sets the ceiling, stacks approach it). No entry = the `[Synergy] ResistMultiplierDefault` backfill | `SpeciesSynergyResist.txt` |
| `synergyBonus` | Per-Synergy-stack payoff for the **player** — a list of `{ "stat": …, "perStack": … }` | `SynergyBonus.json` |

A few tables are **global**, not per-creature: the consumable blankets (`data/blankets.json`) and the
weather-resist foods that grant temperature relief.

### Overriding a shipped table without a rebuild

The one-value-per-species tables — `SpeciesCombatStyle.txt`, `SpeciesSynergyResist.txt`,
`SpeciesSlopeTilt.txt`, `SpeciesCarry.txt`, `SpeciesYawOffsets.txt` — ship embedded in the DLL as the
built-in defaults, and each accepts a **same-format override file dropped next to the configs**:

```
BepInEx/config/SpeciesCombatStyle.txt
```

```ini
# one row per species; an exact key beats the longest name match
Pearlbird=opposite
Armored Hyena=chase
```

A species named in the override file takes that value; a species named in neither layer takes the
built-in default for that axis. The override is re-read live by a dev verb, so retuning costs no
rebuild and no relaunch:

| Table | Override file | Reload verb |
|---|---|---|
| `SpeciesCombatStyle.txt` | `BepInEx/config/SpeciesCombatStyle.txt` | `reloadcombatstyle` |
| `SpeciesSynergyResist.txt` | `BepInEx/config/SpeciesSynergyResist.txt` | `reloadsynergyresist` |
| `SpeciesSlopeTilt.txt` | `BepInEx/config/SpeciesSlopeTilt.txt` | `reloadslopetilt` |

(`SpeciesSynergyResist.txt` has a second override layer above it — the `[Pet]
SpeciesSynergyResistMultipliers` config string, which wins per key. See the
[config reference](./config-reference.md).)

## Item, status & scene names — not raw IDs

You write foods, statuses and scenes as **display names** — `"Raw Meat"`, `"Bleeding +"`,
`"CierzoNewTerrain"` — not numeric IDs. Both are accepted, but names are the readable default; the tool
and the game runtime resolve names → IDs at build/boot.

To make name-checking sharp, `bwspecies check` validates every identifier against snapshots of the
game's real registries in `data/registries/` (`items.json`, `statuses.json`, `scenes.json`), with
"did you mean…" suggestions. Those snapshots are optional (a missing one just skips that check) and are
produced in-game with the **`registrydump`** dev verb, then copied into `data/registries/`. See
`data/registries/README.md` for the capture steps.

## The `bwspecies` CLI

`bwspecies` is a small .NET console tool (`tools/BwSpeciesTool`). Run it from anywhere in the repo:

```sh
./scripts/bwspecies list          # every species + which axes each defines
./scripts/bwspecies show Hyena    # one creature's full data sheet
./scripts/bwspecies check         # validate identifiers + cross-file rules
./scripts/bwspecies build         # regenerate the shipped tables from the manifests
```

| Command | What it does |
|---|---|
| `list` | List every species and the axes it defines. |
| `show <species>` | Print one creature's full data sheet. |
| `add` | Interactively author a new species (menu-driven). |
| `edit <species>` | Interactively edit an existing species. |
| `check [species]` | Validate identifiers and cross-file invariants (e.g. a food-hex meal must be in the diet; taming IDs mustn't collide). |
| `names` | Rewrite any raw numeric IDs in the manifests back to display names (round-trip-safe). |
| `seed` | Fill comfort bands from wiki region data (non-destructive). |
| `build` | Regenerate the embedded shipped tables from the manifests. |
| `build --check` | Report (without writing) whether any shipped table has drifted from the manifests. |
| `ids list` / `ids alloc` / `ids gen` / `ids check` | Inspect, claim, emit and verify the mod's item and status IDs (see below). |
| `import` | Read the shipped tables back into `data/species/*.json` manifests (a one-time bootstrap). |

## IDs are allocated, never typed

Every ID this mod registers — items, recipes, status and effect presets — is drawn from an ID range
allocated to this workspace by the Outward modding community, and recorded in a ledger. The generated
constants, the SideLoader pack XMLs and the item registry are all **written from that ledger** by
`bwspecies ids gen`, and `bwspecies ids check` fails the build if any ID-shaped literal in the tree
isn't accounted for. Adding a new tamable species needs no manual step: its chow and recipe-scroll
IDs are allocated for you.

The practical consequence for authors: **never hand-type a numeric ID into a manifest or into code**.
Write display names (above) and let the tool resolve them.

> **Note for players on old saves.** The ID range moved once. A character created before that move
> loses its Beastwhispering custom items, its learned Beastwhispering skills, and the enchantment on
> any arrows fletched with pet feathers. Tamed pets themselves are unaffected — re-buy the skills
> from the trainer and re-craft the items.

## Adding or editing a creature

1. **Author the manifest.** Either run `bwspecies add` (or `bwspecies edit <species>`) and answer the
   prompts, or hand-write `data/species/<Name>.json`.
2. **Validate.** `bwspecies check <species>` catches typo'd item/status/scene names and cross-file
   rule violations before they ever reach the game.
3. **Regenerate the tables.** `bwspecies build` writes the manifest data into the embedded shipped
   tables.
4. **Build the mod.** `dotnet build Outward.slnx -c Release`, then deploy as usual.

For a guided, creature-by-creature workflow that resolves names to IDs and proposes thematic values
with wiki links, the repo also ships a **`build-beast`** helper.

## Gotcha: `//` vs `#` comments

The JSON axis files accept only **`//`** line comments. A `#` note dropped into a `.json` manifest
breaks that whole generated table at load. The plain-text (`.txt`) axes are the opposite — they accept
`#`. `bwspecies check` and `build --check` lint for the wrong comment flavour and flag it as an error,
so a mistake is caught before it ships rather than silently blanking a table in game.

## See also

- [Beastwhispering overview](./README.md)
- [Creatures](./creatures.md) — the tameable roster these manifests describe
- [Feeding & diet](./feeding-and-diet.md) · [Temperature & blankets](./temperature-and-blankets.md) · [Combat & companion](./combat-and-companion.md)
- [Config reference](./config-reference.md) — the runtime kill-switches for each system
- [Dev verbs](./dev-verbs.md) — `registrydump` and the `reload…` retune verbs
- [Mods index](../README.md)
