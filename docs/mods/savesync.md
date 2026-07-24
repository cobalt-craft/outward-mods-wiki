# SaveSync — sync your saves across devices

**SaveSync** is a mod for Outward that keeps a character's save in sync across the machines you play
on — a desktop, a laptop, a Steam Deck — via a mounted or shared folder. Outward has no built-in way
to do this: Steam Cloud is deliberately unsupported for its saves. SaveSync is for players who play
the same character on more than one device.

**At a glance**
- Type: gameplay mod
- Requires: BepInEx 5 (Mono branch), [ForgeKit](../kits/forgekit.md)
- Config: `BepInEx/config/cobalt.savesync.cfg`
- Commands: `BepInEx/config/SaveSync_cmd.txt`

## For players

SaveSync never changes where the game writes — your save always lands in the normal local folder,
exactly like vanilla. The shared folder you point it at is a **synced replica** it keeps up to date on
either side:

- **At launch**, before the character list loads, it pulls in anything newer from the share, so a
  character you last played on another device shows up here too.
- **While you play**, finished save snapshots push out to the share as they complete, once the mod is
  online and holding the sync lock.
- **No connection to the share?** SaveSync just plays from local, like the mod isn't there, and syncs
  up automatically the next time it can reach the share.

### Setup

1. Mount (or sync) the same folder on every device — a NAS share over NFS/SMB, a mapped Windows
   drive, or a local Syncthing folder all work. Pick something with low latency, since the game's quit
   save writes through it.
2. Run the `syncmarker` command once, on any one device, to mark that folder as the live share (this
   stops an unmounted/disconnected mountpoint — which looks just like an empty folder — from being
   mistaken for an empty-but-connected one).
3. On each device, set `MountPath` to that folder and `Enable` to `true` in the mod's settings. Every
   device using the **same Steam account** lands in the same save folder on the share automatically.

### If two devices genuinely disagree

If the same character was played offline on two devices before either could sync, SaveSync notices
the two histories diverged and **pauses** rather than guessing. `syncstatus` names the character; you
resolve it with the `syncresolve` command:

- `syncresolve local` — keep this device's version.
- `syncresolve share` — take the other device's version.
- `syncresolve both` — keep both: the version you didn't pick is filed into a new save slot instead of
  being discarded.

### Multiple devices at once

Only one device pushes to the share at a time, tracked by a small heartbeat lock file. If you launch
the game on a second device while a first is actively playing, the second still picks up everything
already on the share — it just holds its own new progress locally until the first device finishes.
The lock also recovers on its own from a crash or a lost connection: a lock that's stopped
heartbeating is treated as abandoned, so a dead session can never permanently block another device.

## Settings

`BepInEx/config/cobalt.savesync.cfg`:

| Section | Key | Default | Effect |
|---|---|---|---|
| Sync | `Enable` | `false` | Master switch. Off = vanilla, local-only saves. |
| Sync | `MountPath` | *(empty)* | The shared/mounted folder to sync through. Empty = inert. |
| Sync | `MarkerFileName` | `.outward-sync-root` | Sentinel file that marks the share as actually connected. |
| Sync | `OnMountDown` | `FallbackLocal` | What happens if the share isn't reachable at launch: `FallbackLocal` plays local and warns; `RefuseAndLog` does the same but logs more loudly. |
| Sync | `OnLockHeld` | `FallbackLocal` | What happens to *pushing* while another device holds the lock (pulling always still happens): `FallbackLocal` defers and catches up once it's free; `ForceTake` pushes anyway — only use this if you're sure the other device is gone for good. |
| Lock | `HeartbeatSeconds` | `30` | How often the lock's heartbeat (and the share connection) is refreshed. |
| Lock | `StaleSeconds` | `180` | How long a lock can go without a heartbeat before another device treats it as abandoned. |
| Lock | `DeviceName` | this machine's hostname | This device's name in the lock file. **Must be unique per device.** |

## Commands

Write a command into `BepInEx/config/SaveSync_cmd.txt` and it runs on the next poll (results go to the
log).

| Command | What it does |
|---|---|
| `syncstatus` | Current sync state: mode, local/share paths, lock holder, any paused forks. |
| `syncnow` | Force a reconcile and push right now (a pull still needs a fresh launch). |
| `syncresolve <local\|share\|both> [uid]` | Resolve a paused fork — see *If two devices genuinely disagree* above. |
| `synclock` / `synclock release` | Show who holds the lock / release your own after pushing. |
| `syncmarker` | Stamp the share-liveness marker into `MountPath` — the one-time share setup step. |
| `selftest` | Run the mod's internal checks and log the results. |

## How it works

The game's own save folder is never touched or redirected — SaveSync treats it as the single working
copy and the mount as a mirror it keeps current. Outward saves are immutable, timestamped snapshot
folders, so syncing is really "copy any snapshot folders the other side doesn't have yet" plus knowing
which side is ahead. That last part is the interesting bit: because Outward only keeps the 15 most
recent snapshots per character, the common history between two devices can eventually get pruned away
entirely, so SaveSync can't always tell a clean catch-up from a real divergence just by comparing
snapshot lists. A small counter on the share, bumped every time a device pushes, resolves that: if
only your local copy has moved since the last time you synced, that's a push; if only the share has,
that's a pull; if both have, that's a genuine fork, and SaveSync pauses instead of picking a side for
you.

## For modders

SaveSync is a standalone gameplay mod, not a library — nothing here is meant to be depended on by
another mod. It uses [ForgeKit](../kits/forgekit.md) for its dev-tooling loop (the command channel and
self-test harness) the same way every mod in this family does.

## See also

- [ForgeKit](../kits/forgekit.md) — the dev-tooling kit behind the command channel
- [Installing](../installing.md) — general setup for this mod family
- [Mods index](./README.md)
