# Hireling — recruit a townsperson as a follower

**Hireling** is a mod for Outward that lets you recruit a living NPC — a guard, soldier, quest
character, or merchant body — as a human follower who walks with you and fights at your side. It is
for players who want a humanoid companion, and it is the human-follower counterpart to
[Beastwhispering](./beastwhispering/README.md), which tames beasts.

**At a glance**
- Type: gameplay mod
- Requires: BepInEx 5 (Mono branch), [ForgeKit](../kits/forgekit.md), [CompanionKit](../kits/companionkit.md) (which itself pulls in AggroKit and NetKit)
- Config: `BepInEx/config/cobalt.hireling.cfg`
- Commands: `BepInEx/config/hl_cmd.txt`
- Data: an embedded follower-stats table, overridable at `BepInEx/config/FolkStats.txt`
- Save: your follower, per character, at `BepInEx/config/hl_folk_<character uid>.txt`

## For players

Walk up to a recruitable NPC and press **F4** (or run the `recruit` command). The mod makes a copy
of that NPC as your follower — the original stays where it was. Your follower then trails you across
zones and doors, draws its weapon when a fight starts, and helps you kill what you are fighting.

**Who you can recruit**

- Real, living humanoid NPCs: town guards, soldiers, quest characters, and merchant bodies all
  qualify — even ones with no combat AI of their own.
- **Ambient villagers do not qualify.** The wandering townsfolk that populate a city are a
  lightweight kind of character with no body to puppet, so they cannot be recruited. If the nearest
  thing to you is one of them, the mod says so plainly rather than silently doing nothing.
- **Beasts do not qualify** — those belong to [Beastwhispering](./beastwhispering/README.md).

An NPC must be within **15 m** (configurable) to recruit. You can target a specific one by name with
`recruit <name>`.

**Giving orders**

- `engage` — send the follower onto the enemy you have locked, or, if you have no lock, to help with
  whatever you are fighting.
- `disengage` — stand down: the follower drops the fight and returns to you until you engage again.
- `dismiss` — let the follower go. This tears it down completely.

**Your recruit survives a relaunch (v1 persistence, 2026-07-26 — built, not yet live-verified).**
The follower is remembered per character in `BepInEx/config/hl_folk_<your character uid>.txt`
(written at recruit, on each zone arrival, and when you quit to the menu; `dismiss` deletes it).
On your next launch, the follower re-forms when you are in a scene that holds a live NPC with the
same name — typically the town you recruited them in — and steps back in at the spot they were
last saved. Their stats re-read from the live NPC, exactly like a fresh recruit.

**What it can't do yet**

- **Re-forming needs a live same-named NPC in the current scene.** Reload out in the wilds and the
  follower waits (quietly) until you next enter such a scene. Their gear/stats are re-captured, not
  frozen — a v1 trade-off.
- **One follower at a time.** Recruit again after you have dismissed the current one.
- **In co-op, only the host recruits and dismisses.** A guest who presses the key is told the host
  handles followers; a guest's own reload never re-forms one.

## Features / How it works

A follower's combat readiness comes from two sources, blended together:

- **What was captured off the real NPC** — a genuine guard keeps its own health, speed, and (if it
  was armed) its own weapon damage.
- **The FolkStats table** — a bandit-like baseline so that even an unarmed townsperson fights like a
  capable brawler instead of a helpless civilian.

By default the live capture's health and speed win over the table (a real guard stays as tough as it
looked), while the table fills in fighting stats an unarmed villager would otherwise lack. You can
change that blend in the settings, or turn the table off entirely to use only what was captured.

The follower's body is a clone of the source NPC, so it keeps that character's look, including its
gear. The engine spine that makes it follow, hold position relative to you, and take part in combat
is provided by [CompanionKit](../kits/companionkit.md); Hireling supplies the recruit rules and the
stats layer on top.

## Settings

`BepInEx/config/cobalt.hireling.cfg`:

| Section | Key | Default | Effect |
|---|---|---|---|
| Keys | `RecruitKey` | `F4` | Recruit the nearest recruitable NPC. |
| Recruit | `RecruitRange` | `15` | How close (m) an NPC must be to recruit it. |
| Clone | `EquipmentStrip` | `Components` | How the clone sheds its equipped items. `Components` keeps the dressed look; `GameObjects` is a heavier strip used as a fallback if the lighter one leaves the body broken. |
| Combat | `AttackDamage` | `25` | Damage per hit when neither the capture nor the table supplies a value. |
| Combat | `AttackInterval` | `1.4` | Seconds between the follower's attacks. |
| Combat | `AggroRange` | `12` | How far (m) the follower will engage an enemy you are fighting. |
| Combat | `AttackRange` | `2.6` | The follower's melee reach (m). |
| Stats | `UseFolkStats` | `true` | Blend the FolkStats table with the live capture. Off = capture only. |
| Stats | `PreferCapturedVitals` | `true` | When blending, the captured HP/speed win over the table row. Off = the row sets vitals too. |

### Example configuration

`BepInEx/config/cobalt.hireling.cfg` — created on first launch. Excerpt:

```ini
[Keys]
RecruitKey = F4

[Recruit]
RecruitRange = 15

[Clone]
EquipmentStrip = Components

[Combat]
AttackDamage = 25
AttackInterval = 1.4
AggroRange = 12
AttackRange = 2.6

[Stats]
UseFolkStats = true
PreferCapturedVitals = true
```

**FolkStats table.** The follower-stats table ships embedded with a single bandit-like `Default`
row. To customize it, create `BepInEx/config/FolkStats.txt` in the same format — one row per NPC
name, plus a `Default`. Names are matched exactly first, then by longest partial match, then falling
back to `Default`. Run `reloadfolk` to re-read your changes without restarting. Each row's payload is
a compact stat string (health, speed, impact, resistances, protection, per-type damage); the file's
own header comments spell out every field.

## Commands

Write a verb into `BepInEx/config/hl_cmd.txt` and it runs on the next poll (results go to the log).
Write `help` to list them all.

| Command | What it does |
|---|---|
| `recruit [name]` | Recruit the nearest recruitable NPC, or the nearest one whose name matches. |
| `dismiss` | Dismiss the follower — full teardown. |
| `engage` | Order the follower onto your locked target, or to auto-assist. |
| `disengage` | Order the follower passive — drop the fight and return to you. |
| `folkstatus` | One-glance status: identity, body, anchor HP, stance, effective stats. |
| `folkdump` | Recruit-pipeline dump: nearby candidates with accept/reject reasons, plus a body audit on a live follower. |
| `folkanim` | Animator dump on the follower body (controller, parameters, playing clips). |
| `folkstats` | Stats pipeline: captured vs. table row vs. effective, plus the anchor's live readback. |
| `reloadfolk` | Re-read the `FolkStats.txt` override and re-apply now. |
| `selftest` | Run the mod's internal checks and log the results. |

## For modders

Hireling is **CompanionKit consumer #2** — the reference *human*-follower build, distinct from
Beastwhispering's beast pets. The companion spine (the humanoid puppet body, the welded combat
anchor, follow, and stance) all come from [CompanionKit](../kits/companionkit.md); this mod owns only
the parts that make it Hireling: the recruit policy (which characters qualify and why an ambient
villager is rejected), a human-stats layer (`FolkStats`) that merges a bandit-like table with the
live NPC capture, and the v1 save schema (`Hireling.Core.FollowerSave` — name + last-saved
scene/position). It builds the follower body through the kit's humanoid puppet factory, drives it
through the kit's `Companion` aggregate, and rides the kit's persistence spine
(`CompanionPersistence<FollowerSave>` + a nearby-rung-only `BodyAcquisition` ladder — see the kit
page's *Persistence* section). Its command channel and the config-override table loader come
from [ForgeKit](../kits/forgekit.md).

## See also

- [Beastwhispering](./beastwhispering/README.md) — the beast-taming sibling mod
- [CompanionKit](../kits/companionkit.md) — the companion spine both mods share
- [ForgeKit](../kits/forgekit.md) — the dev-tooling kit behind the command channel and tables
- [Mods index](./README.md)
