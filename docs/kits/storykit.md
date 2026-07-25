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
| `storyreload` | Drop stale references and re-assert placements for the current scene. (Template *registration* stays boot-time — a new NPC needs a relaunch.) |
| `selftest` | Run the StoryKit self-test (`[SELFTEST] PASS/FAIL … DONE`). |
| `storyrecon` / `qeventdump` / `qeventadd` / `qeventset` / `qeventdel` / `qeventage` / `qeventlisten` | Quest-event recon verbs — exploratory tooling, not part of the NPC/trainer surface. |

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
[BepInDependency(StoryKit.Plugin.GUID, BepInDependency.DependencyFlags.HardDependency)]
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

Validate the tree offline in your own tests before it ever reaches the game:

```csharp
var issues = TreeLayout.Validate(spec.Trainer.Tree);
Assert.False(TreeLayout.HasErrors(issues));   // errors would refuse the NPC at build
```

### Public API

| Member | Purpose |
|---|---|
| `NpcRegistry.Register(NpcSpec)` | Register one NPC; call once from your `Awake`, before packs load. |
| `NpcRegistry.Find(id)` | Look up a registered `NpcSpec`. |
| `NpcRegistry.TrySpawn(id)` | Spawn at the spec's placement (master-only, duplicate-safe). |
| `NpcRegistry.SpawnAt(id, pos, rotY)` | Spawn at an explicit position/yaw. |
| `NpcRegistry.Despawn(id)` / `Respawn(id)` | Remove, or remove-and-respawn at the (possibly retuned) placement. |
| `NpcRegistry.StatusDump(idOrNull)` | Log template/placement/live state for one NPC or all. |
| `TreeLayout.Validate(SkillTreeDef)` → `List<TreeIssue>` | Offline layout validation; pair with `TreeLayout.HasErrors(...)`. |

Core data types: `NpcSpec`, `Placement`, `TrainerSpec`, `DialogueSpec`, `Choice` (with
`Choice.Train` / `Choice.Reply` factories), `SkillTreeDef`, `SlotDef`.

### Notes and limitations

- **One placement per NPC.** A spec can hold several `Placement`s, but the director uses the first;
  conditional/rotating placement is not implemented.
- **Trainer NPCs only.** A spec must carry a `TrainerSpec` with a tree; a plain
  dialogue-only NPC is refused at build.
- **No Train choice → unreachable tree.** If the dialogue has no `Choice.Train`, the skill tree can't
  be opened in conversation (StoryKit warns).
- **Registration is boot-time.** An `NpcRegistry.Register` that arrives after packs have loaded is
  rejected — new NPCs need a relaunch. `storyreload` only re-asserts placement, not registration.
- **No quest engine.** Dialogue choices are Train and Reply only; there is no choice kind that fires
  a quest/story event, and no quest state model.

## See also

- [Kits index](./README.md)
- [ForgeKit](./forgekit.md) — the dev-tooling kit StoryKit depends on
- [SkillKit](./skillkit.md) — ship the custom skills a StoryKit tree sells
- [Beastwhispering skills](../mods/beastwhispering/skills.md) — the reference consumer's trainer NPC and tree
- [Wiki home](../README.md)
