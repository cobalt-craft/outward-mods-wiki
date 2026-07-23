# Wiki style guide

These pages document the mods and kits for outside readers — **players** and **modders**. Keep them
factual, present-tense, and scannable. The house style is modeled on the
[Outward Wiki](https://outward.wiki.gg/).

## Do

- **Open with a one-line definition**, subject name in bold: "**Beastwhispering** is a mod for Outward
  that lets you tame wild animals as pets."
- **Lead with the reader.** Player-facing content first; a "For modders" section after. A pure library
  gets a one-line "you won't interact with this directly."
- **Use tables for anything repeatable** — settings (key / default / effect), commands, data columns,
  creature stats. Use bullets for tips and short lists.
- **Present tense, third person, neutral.** Describe what the software does.
- **Cross-link with relative links** (`../kits/forgekit.md`, `./taming.md`). End each page with a
  "See also" and a link back to its section index.
- **Be honest about limitations** where they affect a reader — framed as user-facing limitations, not
  QA state. `kits/spawnkit.md`'s "Known issues" is the model.

## Don't

- **No internal dev-status churn** — no "not live-verified", testplan numbers, "Bug N", build dates as
  status, or refactor history. That belongs in `docs/`, not here.
- **No private data** — no real names, hostnames, IPs, or local machine paths.
- **Don't copy stale plan docs.** Verify config keys, verb names, and API against the actual code
  (`Config.Bind(...)` calls, the verb registry, public types).

## Page template

```
# <Name> — <one-line tagline>

**<Name>** is a <kit|mod> for Outward that <does X>. <who it's for>.

**At a glance**
- Type: reusable library (kit) / gameplay mod
- Requires: BepInEx 5 (Mono branch)<, + deps>
- Config: `BepInEx/config/<guid>.cfg`   (omit if none)
- Commands: `BepInEx/config/<mod>_cmd.txt`   (omit if none)

## For players       <- only where a player-facing surface exists; else a one-line "it's a library"
## Features / How it works
## Settings          <- config table
## Commands          <- dev-verb table
## For modders       <- how to depend on it, public API, extension points, traps
## See also          <- relative links + a link back to the section index
```

## Layout

```
docs/wiki/
  README.md            Home / landing index
  STYLE.md             this file
  installing.md        player install guide
  kits/                one page per reusable library + an index
  mods/                one page (or subfolder) per mod + an index
```
