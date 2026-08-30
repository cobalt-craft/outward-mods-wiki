# StoryKit — add an NPC, trainer, and skill tree

**StoryKit** is a reusable library (kit) for Outward that lets a mod add an **NPC**: a standing
character in the world with a dialogue graph and, if it's a trainer, a full in-game skill tree the
vanilla Trainer window sells from. A consuming mod describes the NPC as plain data; StoryKit builds
the character, spawns it at a fixed spot, wires its conversation, and compiles its skill tree — no
custom UI, no NodeCanvas hand-editing. It's a modder's library.

**At a glance**

- Type: reusable library (kit)
- Requires: BepInEx 5 (Mono branch), [ForgeKit](forgekit.md), and **SideLoader present at runtime**
- Plugin GUID: `cobalt.storykit`
- Config: `BepInEx/config/cobalt.storykit.cfg`
- Commands: `BepInEx/config/StoryKit_cmd.txt`

StoryKit covers **NPCs, trainers, and dialogue**. A quest / story-event engine is **not** part of
the kit — there is no quest authoring API here. Quests remain out of scope for the current library.

## For players

You don't install or interact with StoryKit directly. It arrives as a dependency of a mod that adds
an NPC — for example [Beastwhispering](../mods/beastwhispering/skills.md), whose animal-taming
trainer (Maren) is a StoryKit NPC that sells the mod's pet-skill tree. The NPC and its dialogue are
what you actually meet in-game; StoryKit is the plumbing behind it.

## How it works

A consuming mod calls `NpcRegistry.Register(spec)` once from its `Awake`, passing an `NpcSpec` that
names the NPC, where it stands, what it says, and (for a trainer) the skill tree it sells. StoryKit
then owns the whole lifecycle:

- **Template build** — when SideLoader finishes loading packs, each spec becomes a programmatic
  `SL_CharacterTrainer`. The character is marked *temporary* with no auto-spawn scene, so SideLoader
  never spawns it on its own and never bakes it into a save — StoryKit places it deterministically
  instead, which means a hand-tuned position can never get frozen into someone's save file.
- **Placement** — on every real scene load, if any registered NPC belongs to that scene, StoryKit
  waits for the player to be genuinely in-world (past loading and the void-staging window) and spawns
  the NPC at its placement. Re-entering the scene re-spawns it from the current spec. Spawning is
  duplicate-safe and, in co-op, master-only.
- **Dialogue** — the NPC gets a greeting followed by a multiple-choice menu built from the spec.
  Reply choices show the NPC's answer and loop back to the menu; a **Train** choice reuses Outward's
  own trainer-dialogue flow, so clicking it opens the vanilla training panel. If the dialogue build
  ever fails, the NPC falls back to a minimal greeting-then-train graph rather than becoming
  unresponsive.
- **Skill tree** — the spec's `SkillTreeDef` is validated offline (see below) and compiled into an
  `SL_SkillTree`. From there the **vanilla Trainer / TrainerPanel UI does all the selling** — silver
  costs, prerequisites, and breakthrough gating are the game's own, with zero custom UI.

### The skill tree is validated before the NPC is built

A tree layout is checked against Outward's trainer-UI constraints *offline*, before anything touches
SideLoader. Errors **refuse the NPC at build** (it won't spawn, and the reason is logged); warnings
just log. The rules:

- Rows are **1–5**, columns are **1–3**; no two slots share a cell.
- Every slot has a non-zero `SkillId`, non-negative silver cost, and no duplicate skill in the tree.
- A prerequisite must name a real slot in the same tree (or be `(0,0)` for none); a half-set
  prerequisite is a typo error, and a slot can't require itself.
- There is **exactly one breakthrough slot**, and it sits on **row 3** (the vanilla breakthrough
  row).

Because the check runs on a pure data model with no game references, a consumer can unit-test its
tree layout with no game boot at all.

## Settings

`BepInEx/config/cobalt.storykit.cfg`:

| Section | Key | Default | Effect |
|---|---|---|---|
| `Story` | `EnableStory` | `true` | Master kill-switch. Off: registered specs are still accepted, but no template is built and the director spawns nothing. An NPC already standing in the current scene survives until the next scene change (spawning is the gated act). |
| `Recon` | `EnableStoryRecon` | `false` | Enables the temporary quest-event recon instrumentation and its `[SRECON]` logging. The `qevent*` / `storyrecon` dev verbs are always available but degrade to a note when this is off. Not needed for normal play. |
| `Diag` | `StoryReconPatches` | `false` | Arms the passive recon Harmony taps (quest-event load/save, deaths, interaction triggers) that log a `[SRECON]` timeline. Off keeps those hot paths unpatched. Also gated by `EnableStoryRecon`; a change takes effect on relaunch. |

The `Recon` / `Diag` settings drive scaffolding for exploratory work on a future quest engine and
have no effect on a shipped NPC.

### Example configuration

`BepInEx/config/cobalt.storykit.cfg` — created on first launch. Excerpt:

```ini
[Story]
EnableStory = true

[Recon]
EnableStoryRecon = false

[Diag]
StoryReconPatches = false
```

## Commands

Write a verb into `BepInEx/config/StoryKit_cmd.txt` and it runs on the next poll (even while paused).
Unknown verb or `help` lists them all.

| Verb | Effect |
|---|---|
| `storynpclist` | List every registered NPC — template state, placement, and whether it's live. |
| `storynpcstatus <id>` | Full state for one NPC. |
| `storynpcspawn <id> [here]` | Spawn an NPC at its placement, or `here` to place it two metres in front of you facing you. Master-only. |
| `storynpcdespawn <id>` | Despawn a live NPC. Master-only. |
| `npcmerchanttest [despawn]` | Spawn the built-in test merchant three metres ahead (mobile, tanky, Mefino's backpack, caravan stock, a Shop row), or remove him. Proves the shop path with no consumer mod. Master-only. |
| `storyreload` | Build any templates that were never built (the `EnableStory=false` recovery), drop stale references, and re-assert placements for the current scene. |
| `selftest` | Run the StoryKit self-test (`[SELFTEST] PASS/FAIL … DONE`). |
| `storyrecon` / `qeventdump` / `qeventadd` / `qeventset` / `qeventdel` / `qeventage` / `qeventlisten` | Quest-event recon verbs — exploratory tooling, not part of the NPC/trainer surface. The event-writing ones create **save-baked** quest events under the `bw.srecon.*` prefix; use a throwaway save. |

## For modders

### Depend on the kit

Reference the project with `Private=false` (the DLL ships once, from `dist/StoryKit/`, never copied
into your mod's folder — two copies in `plugins/` means a duplicate-GUID load), and declare the
load-order dependency:

```xml
<ProjectReference Include="..\StoryKit\StoryKit.csproj" Private="false" />
```

```csharp
[BepInPlugin(GUID, NAME, VERSION)]
[BepInDependency(StoryKit.Plugin.GUID, StoryKit.Plugin.VERSION)]   // VERSION = the min-version floor, see kits/versioning.md
```

SideLoader must be present at runtime, but there is **no `[BepInDependency]` on SideLoader** —
StoryKit orders its template build off `SL.OnPacksLoaded` instead.

If the tree sells custom skills, ship them with [SkillKit](./skillkit.md); StoryKit sells any skill
by ItemID and knows nothing about how the skill was defined.

### Define an NPC

Register a spec from your `Awake` — StoryKit loads before you, so the registry is ready. The spec is
pure data from `StoryKit.Core`:

```csharp
NpcRegistry.Register(new NpcSpec
{
    Id   = "mymod.trainer",                 // stable id; also the SL character/template UID
    Name = "Maren",
    Placements = { new Placement("CierzoNewTerrain", x, y, z, rotY: 180f) },
    ChestId = 3000000, BootsId = 3000002,   // optional vanilla-gear ItemIDs for cosmetic dress

    Dialogue = new DialogueSpec
    {
        Greetings = { "You've got the look of someone who's been staring at the wild too long." },
        Choices =
        {
            Choice.Train("train", "Train with me."),            // opens the vanilla trainer panel
            Choice.Reply("who",   "Who are you?", "Maren. …"),  // NPC answers, then loops back
            Choice.Menu("lore",   "Tell me about the beasts.",  // a SUBMENU — see below
                Choice.Reply("lore.hyena", "Hyenas?", "They hunt in threes. …"),
                Choice.Menu("lore.birds",  "Birds?",             // menus nest, up to 3 deep
                    Choice.Reply("lore.pearlbird", "Pearlbirds?", "Loud. Loyal. …"))),
        },
    },

    Trainer = new TrainerSpec
    {
        SkillTreeUID = "mymod.trainer.tree",
        Tree = new SkillTreeDef
        {
            Name  = "My Tree",                                  // trainer-window header
            Slots =                                             // rows 1-5, cols 1-3
            {
                new SlotDef(1, 2, someSkillId, silverCost: 50),
                new SlotDef(3, 2, breakthroughSkillId, 500, breakthrough: true),   // row 3, exactly one
                new SlotDef(4, 1, anotherSkillId, 600, requiredRow: 3, requiredColumn: 2),
            },
        },
    },
});
```

**A plain dialogue NPC** — a lore-giver, a flavour villager — is the same call with `Trainer` left
out entirely:

```csharp
NpcRegistry.Register(new NpcSpec
{
    Id   = "mymod.villager",
    Name = "Old Sten",
    Placements = { new Placement("CierzoNewTerrain", x, y, z) },
    Dialogue = new DialogueSpec
    {
        Greetings = { "Storm's coming. You can smell it." },
        Choices = { Choice.Reply("weather", "How can you tell?", "Forty years on this rock.") },
    },
    // Trainer = null → no skill tree, no trainer panel; everything else is identical.
});
```

Validate the tree offline in your own tests before it ever reaches the game:

```csharp
var issues = TreeLayout.Validate(spec.Trainer.Tree);
Assert.False(TreeLayout.HasErrors(issues));   // errors would refuse the NPC at build
```

### Mobile, combat-capable and merchant NPCs

The same `NpcSpec` can describe a **walking** NPC with an AI, combat stats, a faction, a backpack and
a vanilla **shop**. Everything is opt-in; a spec that sets none of it is the classic pinned talker.

```csharp
NpcRegistry.Register(new NpcSpec
{
    Id = "mymod.pedlar", Name = "Orrin the Pedlar",
    Mobile = true,                            // SideLoader melee AI + NavMeshAgent; NOT pinned
    LookFollowEnabled = false,                // the agent owns yaw on a mobile body
    Faction = "Merchants",                    // Character.Factions by NAME (locale-proof)
    BackpackName = "Mefino's Trade Backpack", // item DISPLAY name; ' and ’ both match
    RandomVisuals = true,                     // rolled face/hair, stable per spec id within a session
    OutfitPool = new List<OutfitSpec>         // one outfit rolled per spawn, pieces by DISPLAY name
    {
        new OutfitSpec("Adventurer Armor", "Adventurer Hat", "Adventurer Boots"),
        new OutfitSpec("Padded Armor", null, "Padded Boots"),
    },
    Ai = new AiSpec { WanderSpeed = 1.1f, CanWanderFar = false, ChanceToAttack = 40f },
    Combat = new CombatSpec
    {
        Health = 900f, Protection = 20f, DamageBonusMult = 0.25f,
        TargetableFactions = new List<string> { "Bandits" },   // whom he will swing at
    },
    Merchant = new MerchantSpec
    {
        StockTableNameContains = "MerchantCaravanTrader",  // the roaming caravanner's regional table
        FallbackItemNames = new List<string> { "Bandage", "Travel Ration" },
        RefreshRateGameHours = 72f,                        // 0 = re-roll on every open
    },
    Dialogue = new DialogueSpec
    {
        Greetings = { "Care to lighten my pack?" },
        Choices =
        {
            Choice.Shop("shop", "Let's see what you're carrying."),   // opens the vanilla shop
            Choice.Reply("road", "Where to?", "Wherever pays."),
        },
    },
});
```

| Field | Effect |
|---|---|
| `Mobile` | Builds a SideLoader melee AI (wander / suspicious / alert / combat states, NavMeshAgent, CharacterAI) and skips the static rig's NoFall, ground-snap and NpcPin. `CharacterStats` stays **enabled**. Move him by driving `AISWander` yourself (e.g. `FollowTransform`). |
| `Ai` | Tuning for that AI: `WanderSpeed`, `CanWanderFar`, `CanBlock`, `CanDodge`, `ChanceToAttack`, `Passive` (targets nobody). Null = SideLoader defaults. Ignored (with a warning) unless `Mobile`. |
| `Combat` | `Health`, `Protection`, `DamageResists[6]` (percent), `DamageBonusMult` (1 = unchanged), `TargetableFactions` (names; null = the faction's default). Requires `Mobile`. |
| `Faction` | `Character.Factions` name. Empty = `NONE` (as before). Unknown name = refused. |
| `BackpackName` | Equipped at spawn; resolved against the live item registry by display name, then prefab name. Unresolvable = refused. |
| `RandomVisuals` | Gender, skin, head, hair style and hair colour are rolled (bounds from the game's own `CharacterVisualsPresets`) instead of SideLoader's all-zero default. Seeded from the spec id + a per-session salt: a respawn in the same session looks the same. |
| `OutfitPool` | A list of `OutfitSpec { ChestName, HelmetName, BootsName }`; one is rolled per spawn on the same seed. Each piece is resolved like `BackpackName`; an unresolvable piece **warns and is skipped** (the spawn proceeds), a resolved one overrides the matching `ChestId`/`HelmetId`/`BootsId`. Empty entries warn offline. |
| `Merchant` | Grafts the game's `Merchant` onto the body: `StockTableNameContains` (a live merchant's table, else a loaded `Dropable` asset), `FallbackItemNames` (generated into the pouch when no table / empty roll), `RefreshRateGameHours`, `NonSavable` (default true), `Buyer`/`Seller`. **Requires `Mobile`** — the graft rides the mobile rig, so a static merchant is refused offline rather than spawning shopless. |
| `Choice.Shop(id, text)` | A root-menu row that opens the shop through a real vanilla `ShopDialogueAction`. Requires `Merchant` (error otherwise). The talk prompt is hidden while the AI is in a combat/suspicious state. |
| `Choice.Action(id, text, replyText, actionId)` | A reply leaf that first runs a host callback registered through `ChoiceActions` — see *Callback rows* below. |

Offline rules (`SpecValidation`): `Combat` without `Mobile`, `Merchant` without `Mobile` and a
`Shop` row without `Merchant` are **errors**; `Ai` without `Mobile`, a `Merchant` with no `Shop` row
and a `Shop` row inside a submenu are warnings. Specs carry no config of their own — the kit's only settings are in
`BepInEx/config/cobalt.storykit.cfg` (see *Settings*); per-NPC numbers belong in the consuming mod's
config.

### Public API

| Member | Purpose |
|---|---|
| `NpcRegistry.Register(NpcSpec)` | Register one NPC; call once from your `Awake`. A registration arriving after pack-load builds its template on demand. |
| `NpcRegistry.TemplatesBuilt` | Has the pack-load template build run? `false` means nothing is spawnable yet. |
| `NpcRegistry.Find(id)` | Look up a registered `NpcSpec`. |
| `NpcRegistry.TrySpawn(id)` | Spawn at the spec's placement (master-only, duplicate-safe). World not live → deferred (below). |
| `NpcRegistry.SpawnAt(id, pos, rotY)` / `SpawnAt(id, pos, rotY, SpawnPolicy)` | Spawn at an explicit position/yaw. **The spawn gate (0.1.7):** while `ForgeKit.Lifecycle.IsGameplayLive()` is false (a load in flight, or the loader still holding gameplay paused — the "press any key" window) a body would come up T-posed / naked, so the registry never spawns into it. `SpawnPolicy.Defer` (the default here) spawns the moment the gate opens (≤30 s, then anyway; abandoned on a scene change); `SpawnPolicy.Refuse` returns false with one `[STORYKIT] … spawn REFUSED — gameplay is not live (…)` line. |
| `NpcRegistry.SpawnAtAndGet(id, pos, rotY)` / `SpawnAtAndGet(id, pos, rotY, SpawnPolicy)` → `Character` | Same, handing back the body (or the already-live one); null = not spawned, reason logged. The default policy is **Refuse** — a caller of this overload needs a synchronous body; wait on `ForgeKit.Lifecycle.WhenGameplayLive` first, or pass `Defer` and accept a null now. |
| `NpcSpec.WalkSpeed` (0.3) / `NpcSpec.RunSpeed` (1.1) | The `AiSpec.WanderSpeed` vocabulary. The default is **`WalkSpeed`** since 0.1.7 — vanilla's townsperson stroll; SideLoader's 1.1 default is a full run and was every "sprinting NPC" sighting. Say `RunSpeed` when you mean it. |
| `[STORYKIT] posture '<id>' @3s: …` (log line) | The posture census, one line per spawned body ~3 s after spawn: `renderers= visualsInit= animator= init= ctrl= aiRoot= closeToPlayer= spawnAnimDone= live=`. A bare body (Bug 46) gets `InitDefaultVisuals`+`Rebind`; an unbound animator gets enabled+`Rebind`; an inactive AI root on a live world is re-activated only when vanilla's own rule would have (spawn anim done, alive, not quest-gated) and otherwise just named. A healthy line on every spawn is what proves the gate held. |
| `NpcRegistry.TryGetLive(id, out Character)` / `TryGetMerchant(id, out Merchant)` | The live body / its grafted vanilla `Merchant`. |
| `NpcRegistry.IsInDialogue(id)` → `bool` | True while the NPC is talking or trading (dialogue tree running, listed in the game's conversation roster, or `Merchant.Buyer` set). A consumer that walks the NPC polls this to stop mid-conversation. |
| `NpcRegistry.Rebuild(id, NpcSpec)` / `ReplaceGreetings(id, greetings)` | Swap the dialogue of a STANDING NPC (greeting variants, new rows) without a despawn. Only the dialogue half is honoured live — body shape (Mobile/Combat/Merchant/Backpack/Faction) is fixed by the template. |
| `NpcRegistry.Despawn(id)` / `Respawn(id)` | Remove, or remove-and-respawn at the (possibly retuned) placement. |
| `NpcRegistry.Unregister(id)` → `bool` | Forget a spec: despawn the body if live, drop the SideLoader template (the UID can be registered again) and remove the entry. For per-spawn specs (DangerousRoads' road merchant). True = a body was live. |
| `NpcRegistry.StatusDump(idOrNull)` | Log template/placement/live state for one NPC or all. |
| `TreeLayout.Validate(SkillTreeDef)` → `List<TreeIssue>` | Offline layout validation; pair with `TreeLayout.HasErrors(...)`. |
| `SpecValidation.Validate(NpcSpec)` → `List<SpecIssue>` | Offline spec-shape validation (trainer optional, inconsistent trainer refused); pair with `HasErrors` / `FirstError`. |
| `ChoiceActions.Register(specId, actionId, fn)` | Register the host callback behind a `Choice.Action` row (see below). `Unregister(specId)` / `TryGet` / `Has` complete the set. |

Core data types: `NpcSpec`, `Placement`, `TrainerSpec`, `DialogueSpec`, `Choice` (with
`Choice.Train` / `Choice.Reply` / `Choice.Menu` / `Choice.Shop` / `Choice.Action` factories), `AiSpec`, `CombatSpec`,
`MerchantSpec`, `OutfitSpec`, `VisualIndices` (`NpcSpec.Visuals` — explicit gender/skin/head/hair/hair-colour
indices; wins over `RandomVisuals`), `SkillTreeDef`, `SlotDef`, `SpecIssue`. Gear by ItemID: `WeaponId`,
`HelmetId`, `ChestId`, `BootsId`, `BackpackId` (`BackpackName` wins when both are set), `ShieldId`. Pure helpers: `OutfitRoll.Pick(pool, roll01)`,
`VisualRoll.Seed(id, salt)` / `Roll(seed, bounds)` / `Roll01(seed, stream)`.

### Ring spawn — several NPCs around a point, collision-safe (`RingSpawner`, 0.1.8)

Register your specs, then put them on a ring in one call. Each slot is navmesh-snapped (Walkable
area, 2.5 m), refused when a non-trigger collider stands at body height or another body was just
placed within `minSpacing`, and re-tried through `StoryKit.Core.RingPlan.Fallbacks` (±15°, ±30°,
then radius ×0.75 / ×1.25) before the spec is given up on. Spawns go through
`NpcRegistry.SpawnAtAndGet(id, pos, yaw, SpawnPolicy.Refuse)`, so the gameplay-live gate applies.

```csharp
public sealed class RingSpawnResult { public List<Character> Spawned; public List<string> Refused; public string Describe(); }
public static class RingSpawner {
    public static RingSpawnResult SpawnAround(IList<string> specIds, Vector3 centre, Vector3 heading, float radius,
        float centreAngleDeg = 180f, float spreadDeg = 120f, float minSpacing = 1.5f, bool faceCentre = true);
}
```

`centreAngleDeg` is measured clockwise from `heading` (a world forward): pass the player's
`transform.forward` and 180 to put the group behind the player, 0 to put it in front.

```csharp
var r = StoryKit.RingSpawner.SpawnAround(new[] { "my_guard_1", "my_guard_2", "my_guard_3" },
    player.transform.position, player.transform.forward, radius: 4f);
Plugin.Log.LogMessage(r.Describe());   // "3 spawned" or "2 spawned, 1 refused [my_guard_3: every candidate blocked (…)]"
foreach (string why in r.Refused) NpcRegistry.Unregister(why.Split(':')[0]);   // per-spawn specs must not linger
```

Pure planner: `RingPlan.Slots(count, radius, centreAngleDeg, spreadDeg)`, `RingPlan.Fallbacks(slot, radius)`,
`RingPlan.Spaced(candidate, placed, minSpacing)` over `RingSlot { AngleDeg, X, Z }`. Log tag `[RING]`.
Not live-verified yet.

### Saved characters — read other saves, build a look-alike, hand items across

Since 0.1.6 StoryKit can read the player's OTHER saved characters (the ones not being played) and
turn one into an NPC that looks like them, and it can move items between such a save and the live
character. First consumers: Echoes (the wandering look-alikes) and DangerousRoads' "familiar
stranger" card. Everything lives in the `StoryKit.Saves` namespace (plugin side) and
`StoryKit.Core` (the pure parser, unit-tested).

| Member | Purpose |
|---|---|
| `SaveScan.Scan(log, warn)` → `List<SavedCharacterRecord>` | Walk `SaveGames/<account>/Save_*/`, load the newest readable snapshot of each character with the game's own `CharacterSave.LoadFromFile` (gzip + XML; safe in-game, no `SaveManager` tables). **Null = the save path is not known yet** (retry later); **empty = known, no saves**. |
| `SavedCharacterRecord` | `Uid`, `Name`, `AreaName`, `Instance` / `SnapshotDir` (the snapshot folder), `IsRetired`, `Hardcore`, `Visuals`, `Items` (every parsed entry), `EquippedItemIds`, `BagUid`, `Offer()` (pouch + bag contents), `IsInUse` (is this the character being played). |
| `SaveScan.IsInUse(uid)` | `SaveManager.IsCharInUse` with the null guards. Always exclude in-use characters from anything that spawns or writes. |
| `SavedCharacterSpec.FromRecord(rec, id, opts)` → `NpcSpec` | The look-alike: the save's five visual indices verbatim (`NpcSpec.Visuals`) and the worn gear resolved to SL slots via each prefab's `EquipSlot` (`HelmetId`/`ChestId`/`BootsId`/`BackpackId`, plus `WeaponId`/`ShieldId` when `opts.ShowWeapons`). Options: `ShowWeapons`, `ShowBackpack`, `Faction`, `Mobile` (default true), `Greeting`. Placement, AI tuning and dialogue rows are yours to add before `NpcRegistry.Register`. |
| `SaveHandoff.Mint(SavedItem, Transform dest)` → `Item` | A live copy of one saved entry: `GenerateItemNetwork` + `ChangeParent` (guest-safe), stack size, durability stamped after `ProcessInit`, enchants via vanilla's first-update queue. |
| `SaveHandoff.OfferPick(rec, offer, taker, onTaken)` | Show the offered entries (typically `rec.Offer()`) to the LOCAL player in the vanilla container panel; the first item moved out fires `onTaken(savedEntry, liveItem)`, closes the panel and destroys the other copies (Take All is hidden, and any extra removal is destroyed regardless). |
| `SaveHandoff.RemoveFromSave(rec, savedEntry, qty)` → `bool` | Write the take back: a NEW snapshot folder `Save_<uid>/<yyyyMMddHHmmss>/` (every file copied, the character file rewritten with the entry removed or its `Quantity` reduced, `PSave.DateTime` bumped, `Manifest.txt` ending `Done`). Refuses while a save is in progress or the character is in use. Cloudward syncs by folder name, so it is an ordinary push. |
| `SavedItemParser` (Core) | `TryParse(xml)`, `WornOf` / `PouchOf` / `BagContentsOf` / `Offer`, `EquippedItemIds`, `EnchantmentIds(subClassesData)`, `NewestFirst`. Hierarchy encodings are vanilla's: `"2"+owner` worn, `"1Pouch_"+owner+";…"` pouch, `"1"+bagUid+"_Content;…"` bag. |
| `SaveSnapshotRules` (Core) | `SnapshotName` / `NextSnapshotName` (always sorts newest), `PSaveStamp`, `EditItemEntry`. |

```csharp
using StoryKit.Saves;

List<SavedCharacterRecord> saves = SaveScan.Scan(m => Log.LogMessage(m), w => Log.LogWarning(w));
if (saves == null) return;                    // save path not ready — try again on the next scene load
foreach (SavedCharacterRecord rec in saves)
{
    if (rec.IsRetired || rec.IsInUse) continue;
    NpcSpec spec = SavedCharacterSpec.FromRecord(rec, "mymod.stranger." + rec.Uid,
        new SavedCharacterSpecOptions { Faction = "Merchants", Greeting = "You seem familiar." });
    spec.Combat = new CombatSpec { TargetableFactions = new List<string> { "Bandits" } };
    NpcRegistry.Register(spec);
    Character body = NpcRegistry.SpawnAtAndGet(spec.Id, pos, yaw);

    // Later, from a Choice.Action callback:
    SaveHandoff.OfferPick(rec, rec.Offer(), player, (saved, live) =>
        SaveHandoff.RemoveFromSave(rec, saved, live.RemainingAmount));
}
```

Log tag for the hand-off: `[SAVEHANDOFF]`. No config keys of its own — the consumer decides what
to offer and when. Everything here is **built, not live-verified** until its consumer's testplan
says otherwise.

### Callback rows (`Choice.Action`) — a reply that knows something

A `Choice.Action(id, text, replyText, actionId)` row is a **Reply leaf that first runs one of your
own callbacks**. If the callback returns a non-empty string it *replaces* the reply text for that
click, so a row can answer with live state the spec never knew — a running tab, a quest step, a
reputation read. An empty (or null) return leaves the authored `replyText` standing.

```csharp
// once, before the NPC is rigged — registration is keyed by SPEC id
StoryKit.ChoiceActions.Register("dr_merchant_7", "balance", player =>
    Debt.Of(player) > 0 ? $"You still owe me {Debt.Of(player)} silver." : "");   // "" = the spec line

var spec = new NpcSpec { Id = "dr_merchant_7", Name = "Caravan Trader", Mobile = true };
spec.Dialogue.Greetings.Add("Road's been quiet. Lucky us.");
spec.Dialogue.Choices.AddRange(new[]
{
    Choice.Shop("shop", "Let's see what you're carrying."),
    Choice.Menu("business", "About our business.",
        Choice.Action("balance", "What do I owe you?", "Nothing at all.", "balance")),
});
```

- **Register BEFORE the NPC is rigged.** A row whose `actionId` has no registered handler is dropped
  at plan time with `[STORYKIT] … is an Action row with no registered callback … dropped` — a row that
  does nothing is worse than an absent one. Registering later takes effect on the next rebuild.
- **Legal at any menu depth**, unlike `Choice.Train` / `Choice.Shop`: each Action row owns its own
  pair of nodes rather than pointing at a singleton. It returns where a plain Reply would (the parent
  menu, or the greeting at the root).
- **`Kind = Action` with an empty `ActionId` is an ERROR** from `SpecValidation`.
- **Your callback may throw.** It is called inside a try/catch: the throw is logged and the
  conversation still reaches the statement with the authored text.
- The argument is the `Character` who is talking, resolved when the row is clicked (never a
  serialized binding), and may be null in odd lifecycle moments — guard it.
- **`NpcRegistry.Unregister(id)` drops the whole spec's callback table**, so a consumer minting
  per-spawn-unique specs (`dr_merchant_<n>`) does not leak one closure per event.

### Nested topic menus (`Choice.Menu`)

A `Choice.Menu(id, text, children…)` row opens a **submenu** instead of speaking a line — the way to
organise an NPC who has a lot to say (a tutorial NPC, a lore index) without a twenty-row list.

- **StoryKit appends the Back row itself.** You never author one, so a menu cannot dead-end. Set
  `Choice.BackText` to relabel it (`"← Never mind."` at the top of a tree reads better than "Back").
- **A leaf returns to the menu it was chosen from**, so a reader browses a topic list instead of
  being thrown to the top. Only a leaf at the ROOT loops back through the greeting.
- **Nesting is capped at 3 levels deep**, and an empty submenu or a `Children` cycle is an ERROR
  from `SpecValidation` — the emitter is recursive, so a cycle would never return. Validate offline.
- **`Choice.Train` only works at the root.** The trainer node is a singleton in the graph; a Train
  row inside a submenu is dropped with a warning.
- Keep a menu at or under about **6 rows** — the dialogue panel numbers rows `1..N` and grows the
  list on demand, so a longer menu functions but reads badly and loses its number hotkeys' clarity.

The whole tree prints as an indented outline at rig time (`[STORYKIT] dialogue built for '<id>'`),
which is the fastest way to check a graph without talking to the NPC. A
`[STORYKIT] … dialogue edge REFUSED for '<choiceId>'` line means that row is dead.

### Notes and limitations

- **One placement per NPC.** A spec can hold several `Placement`s, but the director uses the first;
  conditional/rotating placement is not implemented.
- **A trainer is optional.** Leave `NpcSpec.Trainer` null for a plain dialogue NPC — a lore-giver,
  a flavour villager, a quest-giver stub. It gets the same rig, placement, look-follow, pin and
  ground-snap; it just sells nothing, and any `Choice.Train` in its dialogue is dropped with a
  warning rather than rendered as a dead menu entry. What *is* refused is an inconsistent trainer:
  a `TrainerSpec` with no `Tree` or no `SkillTreeUID` could never open a usable window.
- **No Train choice → unreachable tree.** If a *trainer* spec's dialogue has no `Choice.Train`, the
  skill tree can't be opened in conversation (StoryKit warns).
- **Validate the spec offline.** `SpecValidation.Validate(spec)` (pure `StoryKit.Core`, no game
  needed) returns the same warn/error matrix the kit applies at build; `SpecValidation.HasErrors` /
  `FirstError` give you the refusal reason in your own unit tests.
- **Registration before packs load is the normal path**, and the one to prefer — register from your
  `Awake`. A `Register` that arrives *after* SideLoader's packs have loaded is no longer rejected:
  StoryKit builds and applies that one template on demand, so a mod can add an NPC in response to a
  game event. (SideLoader templates can be applied any time after pack-load.)
- **The kill-switch is recoverable.** Booting with `[Story] EnableStory=false` builds no templates
  at all. Turning it back on and running `storyreload` builds them then and there; a spawn attempt
  made while it's off names the kill-switch explicitly rather than blaming SideLoader.
- **No quest engine.** Dialogue choices are Train, Reply, Menu and Shop; there is no choice kind
  that fires a quest/story event, and no quest state model.
- **Mobile NPCs are not live-verified yet.** The mobile/combat/merchant axis is built and
  unit-tested offline; the in-game shop, walking and combat behaviour await a live session
  (`npcmerchanttest`).
- **The recon verbs write save-baked event UIDs.** `qeventadd` and friends create real quest events
  under the `bw.srecon.*` prefix — a Beastwhispering-flavoured name kept deliberately, since
  renaming it would orphan events already written into existing saves. They are exploratory tooling:
  use them on a throwaway save, not on one you care about.

## See also

- [Kits index](./README.md)
- [ForgeKit](./forgekit.md) — the dev-tooling kit StoryKit depends on
- [SkillKit](./skillkit.md) — ship the custom skills a StoryKit tree sells
- [Beastwhispering skills](../mods/beastwhispering/skills.md) — the reference consumer's trainer NPC and tree
- [Wiki home](../README.md)
