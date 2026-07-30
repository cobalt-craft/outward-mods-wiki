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
| `signatureAttack` | The Hunt as One special: trigger, windup, status, damage mult, cooldown, build-up, `Kind` (Melee / Ranged / Brace), and an optional `taunt` (its minimum and maximum aggro-pin seconds, lerped over loyalty) | `SpeciesSpecialAttacks.txt` |
| `forTheKill` | The For the Kill execute debuff, plus an optional `killBuff` | `ForTheKill.json` |
| `foodHexes` | Which fed foods add which hex build-up on Hunt as One | `FoodHexes.json` |
| `gifts` | The Gift skill's drop table (default drop + loyalty-lerped drops + nothing-chance) | `PetGifts.json` |
| `senses` | Interesting item/creature spawns the pet noses out ("scent") | `PetSenses.json` |
| `scavenge` | Loot containers the pet rolls extra times, and how many per loyalty tier | `PetScavenge.json` |
| `sigils` | How the Hunt as One hit changes while the pet stands in a mage sigil | `Sigils.json` |
| `buffFoods` | Fed consumables that grant a temporary damage buff. Independent of `diet`: an item in both is fed as a meal **and** buffs, off one item; an item only here buffs without feeding | `BuffFoods.json` |
| `skillEchoes` | Per-skill overrides for the pet's bonus strike on your weapon skills | `SkillEchoes.json` |
| `loyaltyGrowth` | How the pet's stats grow with loyalty (per stat group, percent at 100) | `SpeciesGrowth.txt` |
| `donorScenes` / `donorObject` | Which scenes the creature's body can be harvested from | `DonorScenes.txt` (CompanionKit) |
| `yaw` | Rig-facing correction if the model walks sideways/backward | `SpeciesYawOffsets.txt` |

A few tables are **global**, not per-creature: the consumable blankets (`data/blankets.json`) and the
weather-resist foods that grant temperature relief.

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
| `import` | Read the shipped tables back into `data/species/*.json` manifests (a one-time bootstrap). |

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
