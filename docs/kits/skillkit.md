# SkillKit — ship a custom learnable skill

**SkillKit** is a reusable library (kit) for Outward that owns the *mechanism* of shipping a custom
learnable skill — active or passive — so a mod doesn't have to rediscover Outward's skill-wiring
traps. The consuming mod owns the *content* (the skill's SideLoader XML, its ItemID, its icon art,
and what the skill actually does); SkillKit owns the wiring, the icons, the cast animation, the
cooldown, and the convergent re-stamping that keeps a learned skill correct across saves and scene
loads. It's a modder's library.

**At a glance**

- Type: reusable library (kit)
- Requires: BepInEx 5 (Mono branch), [ForgeKit](forgekit.md), and **SideLoader present at runtime**
- Plugin GUID: `cobalt.skillkit`
- Config: none
- Commands: dev verbs registered onto a consumer's command channel (see [Commands](#commands))

## For players

You don't install or interact with SkillKit directly. It arrives as a dependency of a mod that adds
skills — for example [Beastwhispering](../mods/beastwhispering/skills.md), whose pet skills are all
SkillKit registrations.

## How it works

A consuming mod ships an `SL_Skill` XML in its own SideLoader pack (which clones a vanilla donor
skill into a new ItemID) and the icon PNGs as embedded resources in its own DLL. From its `Awake`
the mod calls `SkillRegistry.Register(...)` once per skill. SkillKit then takes over all the timing:

- **Pack-load wiring** — when SideLoader finishes loading packs, SkillKit finds the cloned prefab,
  strips whatever child effects the donor came with, and (for an active skill) attaches its own
  effect on a correctly-named host so the game routes the cast into the skill's activation.
- **Dynamic quickslot icons** — a skill can carry a multi-state icon (the vanilla pistol
  Fire/Reload pattern: the sprite shown reflects a live game-state key) or a plain fixed icon. Both
  ride the same convergent re-stamping so the correct sprite survives saves and quickslot restores.
- **Native cast-animation sync** — a skill can pick its cast animation per equipped weapon, every
  frame, so the game's own activation plays the right animation at press time.
- **Per-skill cooldowns** — a config-driven cooldown value can be stamped onto the prefab and every
  learned instance so a live retune takes effect with no relaunch, plus a next-frame cooldown
  refund helper.
- **Convergence** — the correct icon, cast fields, and cooldown are re-stamped at pack-load, at
  player-ready on every scene load, and on a coarse tick, so a skill rebuilt from a save is right
  again within a tick rather than depending on any single event.

### Active vs passive skills

- An **active** skill has an `OnCast` delegate. It runs synchronously from the game's own activation
  pipeline with the casting character, so it can do anything C# can do — toggle a mode, fire an
  attack, apply an effect.
- A **passive** skill (`Passive = true`, no `OnCast`) is a pure **knowledge token**. SkillKit strips
  the donor's child effects so nothing fires at learn time; the consuming mod's feature code reads
  `player.Inventory.SkillKnowledge.IsItemLearned(itemId)` wherever the passive should change
  behavior. Passives are never quickslotable.

## Commands

SkillKit has no configuration file of its own (`Config: none`) and runs no command channel of its
own. It exposes a dev-verb pack that a consuming mod registers onto its own channel
(`SkillVerbs.RegisterAll(...)`); the verbs then answer on that mod's `BepInEx/config/<mod>_cmd.txt`.

| Verb | Effect |
|---|---|
| `castdump` | Dump the player's current cast state (used to observe a stuck cast). |
| `castclear` | Clear a wedged casting state. |
| `skillverify` | Wire-check every registered skill: prefab resolved, effect host present, icon sprites mapped. Logs PASS/FAIL per skill. |
| `skilldump <name-substring>` | Dump learned skills whose name contains the substring, with their live cast fields. |
| `skillitemdump [name]` | Browse loadable Skill-type item prefabs (useful for picking a donor ItemID). |

## For modders

### Depend on the kit

Reference the project with `Private=false` (the DLL ships once, from `dist/SkillKit/`, never copied
into your mod's folder — two copies in `plugins/` means a duplicate-GUID load), and declare the
load-order dependency:

```xml
<ProjectReference Include="..\SkillKit\SkillKit.csproj" Private="false" />
```

```csharp
[BepInPlugin(GUID, NAME, VERSION)]
[BepInDependency(SkillKit.Plugin.GUID, BepInDependency.DependencyFlags.HardDependency)]
```

SideLoader must be present at runtime, but there is **no `[BepInDependency]` on SideLoader** —
SkillKit hooks `SL.OnPacksLoaded` for its ordering instead.

### Register a skill

Ship an `SL_Skill` XML in your own SideLoader pack (`<YourMod>/SideLoader/Items/...` next to your
DLL) that clones a donor into your ItemID, then register a spec from your `Awake`:

```csharp
SkillRegistry.Register(new SkillSpec
{
    ItemId = 91007003,                     // your id block; matches the XML's New_ItemID
    Label  = "PetCommand",                 // log tag
    OnCast = player => TogglePetCommand(), // what the skill DOES (runs from the native pipeline)

    // OPTIONAL: multi-state quickslot icon. For a plain static icon use
    // Icon = DynamicIcon.Fixed("YourSkill.png").
    Icon = new DynamicIcon
    {
        Select  = () => passive ? "engage" : "disengage",   // state key, polled on every icon sync
        Sprites =                                            // embedded PNGs in YOUR assembly
        {
            ["engage"]    = "PetCommandEngage.png",
            ["disengage"] = "PetCommandDisengage.png",
        },
    },

    // OPTIONAL: native cast animation, re-resolved per equipped weapon every frame.
    CastAnim = player => IsBow(player)
        ? new CastPick("PowerShot", Character.SpellCastModifier.Immobilized)
        : new CastPick("Probe",     Character.SpellCastModifier.Attack),
});
```

For a passive, set `Passive = true`, omit `OnCast` and `CastAnim`, and use
`Icon = DynamicIcon.Fixed("<Name>.png")` — without a fixed icon the clone keeps the donor's icon in
the trainer tree, skill menu, and learn toast.

### Public API

| Member | Purpose |
|---|---|
| `SkillRegistry.Register(SkillSpec)` | Register one skill; call once per skill from your `Awake`. |
| `SkillRegistry.Learn(player, itemId)` | Grant the skill to a character (and re-stamp it immediately). |
| `SkillRegistry.Verify(player)` | Assert every registered skill's wired state; logs PASS/FAIL. Bind to a dev verb. |
| `SkillRegistry.SyncIcons(player)` | Re-stamp icons now — call when an icon's state input flips for same-frame feedback. |
| `SkillRegistry.Find(itemId)` | Look up a registered `SkillSpec`. |
| `SkillCooldowns.Sync(itemId, player, seconds)` | Stamp a cooldown onto the prefab + learned instances; call from your config-reload path. |
| `SkillCooldowns.RefundNextFrame(itemId, player, tag, reason)` | Cancel a just-started cooldown next frame. |
| `SkillVerbs.RegisterAll(registry, log, playerFn)` | Register SkillKit's dev verbs onto your command channel. |

`SkillSpec` fields: `ItemId`, `Label`, `OnCast`, `Passive`, `Icon` (a `DynamicIcon`), `CastAnim`
(a `Func<Character, CastPick>`), `ClearCastAfter` (the stuck-cast self-heal, on by default), and
`ResourceAssembly` (where the icon PNGs are embedded — defaults to your DLL).

### The trap table

These are the Outward gotchas SkillKit encodes so you don't have to. Read them before fighting the
kit — most "why doesn't my skill fire" problems are one of these.

| Trap | What goes wrong |
|---|---|
| **Effect-host name** | Outward buckets a skill effect by its host GameObject's *name*; only an activation-named host lands where the game reads it. SkillKit creates it with the right name — never wire an effect GameObject by hand, or the cast silently does nothing with zero errors. |
| **Stuck cast** | The shared Spark-cast donor never fires its cast-done event for a cloned skill, so the casting flag sticks true forever and *every later cast, from any mod*, silently no-ops. `ClearCastAfter` (default on) self-heals a couple of frames after your effects run. |
| **Prefab vs instance** | The quickslot/cast pipeline may hold either the skill prefab or a learned instance; stamping only one leaves a stale icon or stale cast. Everything SkillKit stamps goes through a loop that touches both. |
| **Custom-icon text** | The quickbar blanks the text label for custom-icon items, so any wording must be baked *into* the sprite. Icon reads are also cached unless the skill is flagged for a dynamic icon, and un-pinned sprites can be unloaded — SkillKit handles both. |
| **Bow cast modifier** | A bow must never get the `Attack` cast modifier (it crashes the charge path). SkillKit enforces this on every cast-anim apply. |
| **Passive donor family** | SideLoader has no passive-skill template — a clone keeps its donor's C# type, so a passive's donor must be a *plain* `PassiveSkill` (e.g. Fitness). The `NeedPassiveSkill` family (Steadfast Ascetic, Slow Metabolism, Efficiency) is **not** a `PassiveSkill` and errors once stripped. SkillKit hard-asserts the family at runtime and refuses a wrong donor rather than half-arming it. |
| **No global cooldown** | Outward has no global cooldown — every skill gates only itself. A cooldown of `1` is a double-press debounce; `0` means none. |

### SL_Skill XML template

Ship this in `<YourMod>/SideLoader/Items/<SkillName>/<SkillName>.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<SL_Skill xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema">
  <EffectBehaviour>NONE</EffectBehaviour>
  <Target_ItemID>8200040</Target_ItemID>   <!-- a vanilla donor skill to clone -->
  <New_ItemID>91007003</New_ItemID>        <!-- your ItemID block -->
  <Name>Command Pet</Name>
  <Description>What the tooltip says.</Description>
  <IsPickable>false</IsPickable>
  <IsUsable>false</IsUsable>
  <CastType>Spark</CastType>               <!-- overridden live if the spec sets CastAnim -->
  <CastModifier>Immobilized</CastModifier>
  <Cooldown>1</Cooldown>                   <!-- self-only; 1 = debounce, 0 = none -->
  <StaminaCost>0</StaminaCost>
  <ManaCost>0</ManaCost>
</SL_Skill>
```

For a passive, clone a plain `PassiveSkill` donor (Fitness, ItemID `8205040`) instead, set
`<EffectBehaviour>DestroyEffects</EffectBehaviour>`, `Cooldown` to 0, and omit the costs and cast
fields.

### Icon authoring

Icons are 160×160 PNGs with any wording baked into the sprite. Embed them in your own DLL:

```xml
<ItemGroup>
  <EmbeddedResource Include="Icons\*.png" />
</ItemGroup>
```

Sprite names are suffix-matched against your assembly's resource manifest, so folder prefixes don't
matter.

## See also

- [Kits index](./README.md)
- [ForgeKit](./forgekit.md) — the dev-tooling kit SkillKit depends on
- [StoryKit](./storykit.md) — build a trainer that sells a tree of these skills
- [Beastwhispering skills](../mods/beastwhispering/skills.md) — a mod that ships several SkillKit skills
- [Wiki home](../README.md)
