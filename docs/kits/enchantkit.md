# EnchantKit — apply and spawn enchanted gear for test sessions

**EnchantKit** is a kit (reusable library) for Outward that lets a modder or tester put vanilla
**enchantments** onto gear from a command — enchanting items already owned or worn, or spawning gear
that is already enchanted. It exists to stage enchanted-gear scenarios in a live test session without
running the whole enchanting-table loop (incense, pillars, and the region/weather/time conditions
vanilla enchanting recipes require). It has no player-facing UI.

**At a glance**
- Type: reusable library (kit) — dev / testing tooling
- Requires: BepInEx 5 (Mono branch), [ForgeKit](forgekit.md)
- Config: `BepInEx/config/cobalt.enchantkit.cfg`
- Commands: `BepInEx/config/EnchantKit_cmd.txt`

## For players

EnchantKit is a modder-and-tester tool, not a play feature — it adds no items, quests, or mechanics
to a normal game; you only need it if a project you're testing asks for it.

## Features / How it works

EnchantKit works entirely through vanilla enchantment plumbing — `Equipment.AddEnchantment` and the
game's two enchantment registries — so it ships no SideLoader dependency and installs no Harmony
patches. It does three jobs:

- **Enchant gear in place** — apply one or more enchantments to the equipped main-hand weapon, a worn
  armor slot, or an item sitting in the pouch or backpack. This is a direct apply on an item that is
  already live and owned.
- **Spawn gear already enchanted** — generate an item and land it in the pouch, backpack, on the
  ground, or straight onto the character, with its enchantments already attached. Freshly spawned
  items route their enchantments through the game's own deferred first-update queue so the stats fold
  in correctly once the item finishes initializing (a direct apply on a not-yet-initialized item
  would record the enchantment but leave its bonus unapplied).
- **Enumerate the registry** — list every enchantment the game knows, with its display name, id,
  compatible-gear summary, and incense, and write that catalog to a file.

Two guard rails shape how it behaves:

- **Registration health is the one hard refusal.** An enchantment id must exist in *both* the prefab
  registry and the recipe registry, or the apply is refused with a clear message rather than silently
  doing nothing or throwing.
- **Compatibility is a warning, not a block.** Because a testing tool must be able to probe odd
  combinations (say, a weapon enchantment on armor), an incompatible pairing warns but still applies;
  prefixing the enchantment with `force` silences the warning.

Enchantments applied to real gear persist the way vanilla enchantments do (they ride the item's
saved extra-data), so they survive a save and reload.

### Enchantment names and ids

Every enchantment has a numeric **PresetID**, and those ids exist only inside the game's assets — no
file in the repository lists them. Commands accept an enchantment's display **name** or its PresetID
interchangeably; to get the numbers, load a save and run `enchantregistry` once to write the
name-to-PresetID catalog, or run `enchantlist` to print it. Names are localized and not always
unique, so when two enchantments share a name the lowest PresetID is chosen.

### Known limitations

- **Spawning enchanted gear is one item at a time.** The spawn command always makes a single item,
  and enchanted ammunition **stacks** are out of scope — prefer non-stackable gear (weapons, armor)
  when staging tests.
- **Multiplayer propagation is not guaranteed.** The game authors enchantment state on the session
  master. Applying an enchantment as a non-master client logs a warning and relies on the item's
  own change-sync reaching the other players; whether a guest's enchant reaches the host is not
  guaranteed.

## Settings

`BepInEx/config/cobalt.enchantkit.cfg`, generated on first launch. EnchantKit has a single setting.

| Section | Key | Default | Effect |
|---|---|---|---|
| `Enchant` | `EnableEnchantKit` | `true` | Kill-switch. When `false`, the mutating verbs (`enchant`, `unenchant`, `giveenchanted`) refuse; the read-only diagnostics (`enchantlist`, `enchantdump`, `enchantregistry`) keep working. Live-flippable with `reloadcfg`. |

### Example configuration

`BepInEx/config/cobalt.enchantkit.cfg` — created on first launch. Excerpt:

```ini
[Enchant]
## Kill-switch: false refuses enchant/unenchant/giveenchanted (the mutating verbs).
## The diagnostics (enchantlist/enchantdump/enchantregistry) keep working. Live via reloadcfg.
EnableEnchantKit = true
```

## Commands

Write a verb into `BepInEx/config/EnchantKit_cmd.txt` and it runs on the next poll (even while the
game is paused). Unknown verb or `help` lists them all. Results are logged with the `[ENCH]` tag.

The one delimiter that keeps space-containing item **and** enchantment names unambiguous is **`+`**:
it separates the item from the enchantment list, and one enchantment from the next. An enchantment
reference is a name or a numeric PresetID, optionally prefixed with `force` to skip the
compatibility warning.

| Verb | Does |
|---|---|
| `enchant <slot> <ench>` | Enchant the item worn in a slot (`weapon`, `offhand`, `helmet`, `chest`, `legs`, `boots`, `hands`, `backpack`, `quiver`). |
| `enchant <item> + <ench> [+ <ench>…]` | Enchant a named inventory item (name or ItemID), applying one or more enchantments. |
| `enchant <ench>` | Bare form — enchant the equipped main-hand weapon. |
| `unenchant <slot\|item>` | Strip every enchantment from a worn slot or an inventory item. |
| `unenchant <slot> <ench>` / `unenchant <item> + <ench>` | Remove one named enchantment, leaving the rest. |
| `giveenchanted [pouch\|bag\|ground\|equip] <item> + <ench> [+ <ench>…]` | Spawn an item already enchanted (destination default `pouch`, quantity 1). `bag` refuses with nothing spawned if no backpack is worn. |
| `enchantlist [filter]` | Enumerate the enchantment recipe registry — PresetID, RecipeID, name, compatible-gear summary, incenses; optional name/id filter. |
| `enchantdump` | Per held or equipped item: active enchantment ids and names, live-vs-prefab weapon damage, and registration health. |
| `enchantregistry` | Write `BepInEx/config/enchantkit_registries/enchantments.json` — the offline name-to-PresetID catalog. |
| `selftest` | Zero-interaction environment and grammar checks (`[SELFTEST] PASS/FAIL … DONE`). |

`enchant` prefixes an enchantment with `force` to suppress the compatibility warning, and the
inventory and spawn forms accept several enchantments joined with `+`. Examples:

```
enchant weapon Fire Damage
enchant Iron Sword + force Frost Enchantment + Health Regeneration
giveenchanted equip Iron Sword + 100
unenchant weapon
enchantlist Fire
```

EnchantKit also registers ForgeKit's full **CommonVerbs** pack on this same channel
(`give` / `drop` / `equip` / `teleport` / `goto` / `killnearest` / `learnskill` / … ), so a test
session can stage the surrounding scenario — the gear, the enemy, the position — right beside the
enchant commands.

## For modders

EnchantKit is a self-contained dev-and-test utility rather than a library you build features on: its
apply, spawn, and removal mechanics are internal, so there is no public enchantment API to call. You
add it to a mod's install when you want to stage enchanted gear during test sessions — most usefully
alongside [Beastwhispering](../mods/beastwhispering/README.md), to exercise how its systems interact
with enchanted weapons and armor.

Install it the usual way — a project reference plus a hard dependency so BepInEx loads it (and
ForgeKit under it) first:

```csharp
[BepInPlugin(GUID, NAME, VERSION)]
[BepInDependency(EnchantKit.Plugin.GUID)]   // "cobalt.enchantkit"
public class Plugin : BaseUnityPlugin { … }
```

The mechanics it demonstrates are worth knowing if you enchant items yourself: gate every apply on
both registries being populated for the id; route enchantments onto a freshly spawned item through
the engine's deferred first-update queue rather than a direct call, so the bonus folds in after the
item initializes; and treat compatibility as advisory. The enchantment PresetIDs are unknowable
offline — `enchantregistry` is how you turn them into a catalog you can script against.

## See also

- [Kits index](./README.md)
- [ForgeKit](forgekit.md) — the dev tooling EnchantKit's command channel and shared verbs come from
- [NetKit](netkit.md) — the other SideLoader-free, CI-buildable kit it mirrors in shape
- [Beastwhispering](../mods/beastwhispering/README.md) — the mod its test scenarios most often target
- [Wiki home](../README.md)
