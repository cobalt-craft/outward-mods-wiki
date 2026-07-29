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

**Declining a cast.** By the time your delegate runs, the game has already charged the cost, played
the animation and started the cooldown — so "this cast should not have counted" has to be said
explicitly. Use `OnCastResult` (a `Func<Character, CastResult>`) instead of `OnCast`:

```csharp
OnCastResult = player =>
{
    if (NoPet) return CastResult.Refund;          // clears the cooldown next frame + logs one line
    if (OnCooldownElsewhere) return CastResult.RefuseSilently;   // cooldown stands, kit says nothing
    DoTheThing(player);
    return CastResult.Landed;
},

// OPTIONAL: let the kit own the cooldown instead of the XML. Read fresh each sync, so a config
// edit + reloadcfg retunes the live skill within ~2s — no relaunch, no consumer-side wiring.
CooldownSeconds = () => MyConfig.GiftCooldownSeconds.Value,
```

`OnCast` keeps working exactly as before; set one or the other, not both (the kit warns and
`OnCastResult` wins).

For a passive, set `Passive = true`, omit `OnCast`/`OnCastResult` and `CastAnim`, and use
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

| `SkillRegistry.SyncCooldowns(player)` | Re-stamp every spec's `CooldownSeconds`; also runs on the kit's own ~2s tick. |

`SkillSpec` fields: `ItemId`, `Label`, `OnCast`, `OnCastResult` (the refusable form), `Passive`,
`Icon` (a `DynamicIcon`), `CastAnim` (a `Func<Character, CastPick>`), `CooldownSeconds` (a
`Func<float>` — kit-owned, live-retunable), `ClearCastAfter` (the stuck-cast self-heal, on by
default), and `ResourceAssembly` (where the icon PNGs are embedded — defaults to your DLL).

`SkillRegistry.Init(log, coroutineHost)` is first-caller-wins: SkillKit's own plugin wires it (it is
every consumer's hard dependency, so it always loads first), and a later `Init` logs
`Init ignored — already wired by …` and discards its arguments. That means a second consumer's kit
lines are logged under the first one's mod name — expected, and now said out loud.

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

Whatever loads a PNG at runtime must call **`SkillKit.IconPin.Pin(...)` on BOTH the `Texture2D` and
the `Sprite`**. An unpinned sprite is collected by `Resources.UnloadUnusedAssets` (which mods
themselves call mid-session), and a dead status sprite does not fall back to "no art" — it falls
back to the clone donor's badge, so a pet-hunger indicator silently becomes vanilla *Bleeding*.
That invariant was learned twice independently; it is stated once, in `IconPin`.

## Custom status effects — `SlStatus`

Ship an indicator/timer status of your own (a HUD badge for your mod's state, a stacking buff, a
timed boon) by cloning a plain vanilla status. Added 2026-07-26, lifted out of Beastwhispering,
which had three of these; it lives here because it is deeply SideLoader-typed, exactly like the
skill registration this kit already owns. **Built, not yet live-verified in kit form** — behaviour
is the shipped BW mechanism verbatim (regression row: `docs/status-icons-testplan.md` V15).

Bind the consumer context once at boot, then register from your `SL.OnPacksLoaded` handler:

```csharp
SlStatus.Log = Logger;                                  // warnings name YOUR mod
SlStatus.DefaultIconLoader = (png, tag) =>              // the PNGs live in YOUR DLL
    IconPin.Pin(CustomTextures.CreateSprite(myTexture(png), CustomTextures.SpriteBorderTypes.NONE));

var spec = new SlStatusSpec {
    Id = "MY_Badge", NumId = 91009900, Name = "Watched", Description = "Something is watching.",
    Lifespan = -1f,                    // -1 = permanent indicator; > 0 = a real countdown
    IconPng = "MyBadge.png", IsMalus = true,
    Tag = "[MYMOD]", DisabledSuffix = "the badge is disabled.",
    // BindFamily = <SL_StatusEffectFamily>  // REQUIRED for a STACKING status — see below
};
if (SlStatus.ResolveDonor(spec, out string donor))
    SlStatus.Register(spec, donor, out StatusEffect prefab);
```

| Member | Purpose |
|---|---|
| `SlStatus.ResolveDonor(spec, out donor)` | Probe `DonorCandidates` (default Bleeding→Burning→Poisoned). Call ONCE per set and reuse the donor, so a miss warns once, not N times. |
| `SlStatus.Register(spec, donor, out prefab)` | Clone + strip + stamp. Silent on success (log your own evidence line); `false` = the prefab was missing after `ApplyTemplate`. |
| `SlStatus.Live(player, id)` | The player's live instance, or null. **Also converges the icon** — every read path goes through here. |
| `SlStatus.StackCount / AddStack / Remove` | The obvious ops on the live instance. |
| `SlStatus.RefreshAll(player, id, seconds)` | Reset every running stack's clock so the set expires together (vanilla decays per stack). |
| `SlStatus.SetStacks(player, id, n, seconds, canAdd)` | Converge to exactly `n` stacks, then refresh. `canAdd` = your registration latch. |
| `SlStatus.AddOrRefresh(player, id, seconds)` | Single-instance grant, idempotent. |
| `SlStatus.SyncDuration(id, seconds)` | Live retune: re-stamp the duration onto the prefab. |
| `SlStatus.IconHeld(id)` / `IconState(id, live)` | Boot-line and dump-verb readbacks — `use=False` IS the reason a player sees the donor badge. |
| `SlStatus.EnsureIcon(id, live, tag)` | Re-assert the art; call it from your tick when you already hold the instance. True only when it actually repaired something. |
| `SlStatus.ResetIconRetries()` | Clear the negative cache. Wire it to your config-reload verb. |

What the kit does for you, and why each one is load-bearing:

| Guard | What it prevents |
|---|---|
| The FX strip runs **unconditionally** inside `Register` (not a spec option) | A clone inherits its donor's `FXPrefab`, which is instantiated but never bound to a mesh — one `zero surface area` log line **per frame for the status's whole life** (measured: 111 k lines in one session). It also closes the `FxInstantiation`/`SpecialFXPrefab` back doors, and refuses to strip if SideLoader handed back the SHARED donor prefab. |
| `Tags = new string[0]` | Without it the clone inherits the donor's tags and `StatusEffect.Start()` re-derives `m_defaultStatusIcon` from them: a missing icon then renders as *Bleeding*, not as nothing. Cleared, failure degrades to a blank slot. |
| Icon stamp is **convergent**, into BOTH `OverrideIcon` and `m_defaultStatusIcon` | The prefab-side stamp does not survive `Start()` on live instances, and a sprite can be destroyed mid-session. Repairs self-heal within a tick. |
| Bounded icon retry + negative cache | A genuinely missing PNG would otherwise warn on every read (`Live` is on the read path) forever. It gives up per id, loudly, once — and `ResetIconRetries` re-arms it after a rebuild. |
| `ActionOnHit = None` | The Bleeding donor reduces its own stack when hit; an indicator must not. |
| `BindFamily` + `FamilyMode.Bind` | Without a bound StackAll family, SideLoader auto-creates a `MaxStackCount=1` family for a renamed clone and a second grant OVERRIDES instead of stacking. |

## See also

- [Kits index](./README.md)
- [ForgeKit](./forgekit.md) — the dev-tooling kit SkillKit depends on
- [StoryKit](./storykit.md) — build a trainer that sells a tree of these skills
- [Beastwhispering skills](../mods/beastwhispering/skills.md) — a mod that ships several SkillKit skills
- [Wiki home](../README.md)
