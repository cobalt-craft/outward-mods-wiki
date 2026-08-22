# Combat & companion — fighting at your side

A tamed pet is a real combatant. It joins your fights, keeps the tamed creature's own attributes, and
can be taught signature skills. A native **Companion** tab in the character menu shows its full sheet.

## How your pet fights

- **Auto-assist.** When you engage an enemy, the pet engages it too, chasing and attacking what you
  fight. It falls back to following you when the fight ends.
- **Species-true stats.** The pet keeps the tamed creature's own resistances, protection, health,
  natural attack, and movement speed — a Hyena hits and moves like a Hyena. Those stats are scaled by
  loyalty, so a Devoted pet is meaningfully tougher and stronger than a Broken one. (Species weaknesses
  carry over too — devotion shrinks them, neglect deepens them.)
- **Signature attack — "Hunt as One."** A learnable skill that fires the pet's signature attack while
  you land a synced strike on the same target. Each species' signature is different: a status-inflicting
  bite, a bracing counterattack, a ranged bolt, and so on (see [Creatures](./creatures.md)). A bracing
  species also **taunts** as the stance opens, pinning its current target's aggro onto the pet for
  between one and five seconds depending on loyalty — a deliberate way to pull a dangerous enemy off you.

Combat ranges (how far the pet will engage, its melee reach, and how far it leashes while fighting) are
all tunable in the [config reference](./config-reference.md) (`[Combat]`).

## Commanding your pet

The **Command Pet** skill cycles through three orders — **Engage → Follow → Stay → Engage** — and its
quickslot icon always shows the order your *next* press gives:

- **Engage** — send the pet at your locked target (at any range), or at the nearest enemy in a cone in
  front of you if you have no lock.
- **Follow** — the pet stands down, stops scanning for targets, and returns to following you; the
  encounter actually ends (combat music stops).
- **Stay** — the pet stays passive *and* holds the spot it is standing on. It stays put until you give
  another order, or until you walk far enough away (the normal leash distance, `[Pet] LeashDistance`)
  that it teleports to you — at which point it is **following** again and the icon updates on its own.
  Recalling it (F9) or changing area ends a Stay the same way.

A staying pet does not defend itself — it is as passive as a following one. In co-op, a guest's staying
pet shows as *following* on other players' screens (a cosmetic limit of the current stance mirror).

Command Pet, Hunt as One, and the other pet abilities are learnable skills — see [Skills](./skills.md).

## The damage governor

By default, a pet only reaches its **full** damage once its owner has learned the **Wild Unknown**
skill. Until then, every hit the pet deals is scaled down (to **50%** by default) — a deliberate
progression gate. Learning Wild Unknown removes the penalty. This is separate from, and stacks with,
the global and per-species damage-scale settings used for balance tuning. See
[Skills](./skills.md) and the [config reference](./config-reference.md) (`[Combat]`).

## The Companion tab

Beastwhispering adds a **Companion** tab to the character menu, alongside Inventory, Skills, and the
rest. It shows the pet's full sheet in one place:

- **Vitals** — health, and any over-time recovery in progress.
- **Bond** — loyalty, current tier, and how far to the next tier.
- **Needs** — hunger, comfort (what it's wrapped in / sipping), and overall mood.
- **Combat** — its damage and signature attack, its live resistances, and its current combat state.
- **Bond buffs** — what the bond is currently granting you.

With no pet, the tab shows a styled empty state. Cycle to it with the menu's tab controls like any
other tab.

### Naming your companion

The header at the top of the tab — where your pet's species is shown — **is** the name field.
Click it (or press the pad's top-right face button, **Y** on an Xbox layout) and type a name;
**Enter**, or **A** on a pad, confirms it. **Escape** or **B** cancels and changes nothing.
Clicking away elsewhere keeps what you typed.

Once named, your companion is called by that name everywhere: the tab header, the caption above
its health bar, and every message about it — feeding, gifts, bandaging, crossing into a new
region. Confirming an **empty** name removes it and the pet goes back to showing its species.
Names are capped at 20 characters.

In co-op the name travels with your pet, so other players' game logs identify it the way you do.

Turn the whole thing off with `[PetPanel] EnableRename = false` in
`BepInEx/config/cobalt.beastwhispering.cfg` — see the
[config reference](./config-reference.md#petpanel--companion-tab-controller-nav-and-naming).

## Known issues & limitations

- **The pet is not a second character.** It has no gear slots, no skills of its own to assign, and no
  inventory; everything it does comes from its species data and the skills *you* learn.
- **Ranged signature attacks are only reachable on species you can't yet tame.** The tameable roster's
  signatures are all melee, Brace or hex bites — see [Creatures](./creatures.md).
- **Sigil synergies, worn-gear effects, the scavenge bonus and the damage-scale settings are newer,
  lightly-exercised systems.** They are all data-driven and each has its own kill-switch in the
  config, so if one behaves oddly you can turn just that system off.
- **In co-op, some effects the host resolves show no floating damage numbers on a guest's screen**,
  and a guest's Companion tab reads "no anchor" on the anchor-health rows. The damage itself lands.

## See also

- [Skills](./skills.md) — Hunt as One, Command Pet, Wild Unknown, and the rest
- [Loyalty & bonds](./loyalty-and-bonds.md) — how loyalty scales the pet's stats
- [Creatures](./creatures.md) — each species' signature attack
- [Config reference](./config-reference.md) — `[Combat]` tuning
- [Beastwhispering overview](./README.md)
