# Cloudward — sync your saves across devices

**Cloudward** is a mod for Outward that keeps a character's save in sync across the machines you play
on — a desktop, a laptop, a Steam Deck — via a mounted or shared folder. Outward has no built-in way
to do this: Steam Cloud is deliberately unsupported for its saves. Cloudward is for players who play
the same character on more than one device.

**At a glance**
- Type: gameplay mod
- Requires: BepInEx 5 (Mono branch), [ForgeKit](../kits/forgekit.md)
- Config: `BepInEx/config/cobalt.cloudward.cfg`
- Commands: `BepInEx/config/Cloudward_cmd.txt`

## For players

Cloudward never changes where the game writes — your save always lands in the normal local folder,
exactly like vanilla. The shared folder you point it at is a **synced replica** it keeps up to date on
either side:

- **At launch**, before the character list loads, it pulls in anything newer from the share, so a
  character you last played on another device shows up here too.
- **While you play**, finished save snapshots push out to the share as they complete, once the mod is
  online and holding the sync lock.
- **No connection to the share?** Cloudward just plays from local, like the mod isn't there, and syncs
  up automatically the next time it can reach the share.

### Setup

1. Mount (or sync) the same folder on every device — a NAS share over NFS/SMB, a mapped Windows
   drive, or a local Syncthing folder all work. Pick something with low latency, since the game's quit
   save writes through it.
2. Launch Outward once so the mod writes its config file, then quit. The file is
   `BepInEx/config/cobalt.cloudward.cfg` — **edit it only while the game is closed** (Outward rewrites
   it on quit).
3. In that file, set two keys under `[Sync]` — `Enable` and `MountPath`:

   ```ini
   [Sync]
   Enable = true
   MountPath = Z:/mnt/nas/outward
   ```

   **Path format:** Outward runs under Proton on Linux/Steam Deck, so `MountPath` is a Wine drive path —
   `Z:` is the filesystem root `/`, so a share at `/mnt/nas/outward` is `MountPath = Z:/mnt/nas/outward`.
   On native **Windows** use a normal path or UNC, e.g. `MountPath = Z:\OutwardSync` or
   `MountPath = \\NAS\OutwardSync`. Every device using the **same Steam account** lands in the same save
   folder on the share automatically.
4. Relaunch. Optionally set `[Sync] ForkResolution` (e.g. `ForkResolution = ThisDevice`) to choose how
   conflicts resolve — the default `NewestWins` is usually right.

That's the whole setup — there's no command to run. Cloudward marks the folder as the live share for
you the first time it runs (a small marker file that stops an unmounted/disconnected mountpoint —
which looks just like an empty folder — from being mistaken for a connected one). If you'd rather bless
the share by hand, set `AutoCreateMarker` to `false` and run the `syncmarker` command once instead.

### If two devices genuinely disagree

If the same character was played offline on two devices before either could sync, Cloudward resolves
it automatically per the `[Sync] ForkResolution` setting — and **always backs up the version it
doesn't keep** to `.cloudward-forks/`, so nothing is discarded:

- **`NewestWins`** *(default)* — keep whichever device has the most recent in-game save.
- **`ThisDevice`** — always keep this device's version.
- **`OtherDevice`** — always take the other device's version.
- **`KeepBoth`** — keep this device's version live and preserve the other. *(v1 files it as a
  restorable backup; a true second save slot in the load list is planned for v2.)*
- **`AskMe`** — don't guess; wait for you to decide with the `syncresolve local|share|both` command
  (`syncstatus` names the paused character).

`syncresolve` is also always available as a manual override under any policy.

### Multiple devices at once

Only one device pushes to the share at a time, tracked by a small heartbeat lock file. If you launch
the game on a second device while a first is actively playing, the second still picks up everything
already on the share — it just holds its own new progress locally until the first device finishes.
The lock also recovers on its own from a crash or a lost connection: a lock that's stopped
heartbeating is treated as abandoned, so a dead session can never permanently block another device.

## Settings

`BepInEx/config/cobalt.cloudward.cfg`:

| Section | Key | Default | Effect |
|---|---|---|---|
| Sync | `Enable` | `false` | Master switch. Off = vanilla, local-only saves. |
| Sync | `MountPath` | *(empty)* | The shared/mounted folder to sync through. Empty = inert. |
| Sync | `MarkerFileName` | `.outward-sync-root` | Sentinel file that marks the share as actually connected. |
| Sync | `AutoCreateMarker` | `true` | Create that marker automatically so you only need to set `MountPath` (no `syncmarker` step). It's created once per folder and remembered, so a later launch with the share disconnected is correctly read as offline (not re-marked). A folder that already holds real save data always gets its marker restored. Set `false` for strict manual setup (`syncmarker` required). |
| Sync | `OnMountDown` | `FallbackLocal` | What happens if the share isn't reachable at launch: `FallbackLocal` plays local and warns; `RefuseAndLog` does the same but logs more loudly. |
| Sync | `OnLockHeld` | `FallbackLocal` | What happens to *pushing* while another device holds the lock (pulling always still happens): `FallbackLocal` defers and catches up once it's free; `ForceTake` pushes anyway — only use this if you're sure the other device is gone for good. |
| Sync | `ForkResolution` | `NewestWins` | How a character played offline on two devices is auto-resolved: `NewestWins` = most recent in-game save; `ThisDevice` / `OtherDevice` = always this / the other device; `KeepBoth` = keep this device's live and preserve the other (v1: a restorable backup); `AskMe` = wait for a manual `syncresolve`. The version not kept is always backed up. |
| Lock | `HeartbeatSeconds` | `30` | How often the lock's heartbeat (and the share connection) is refreshed. |
| Lock | `StaleSeconds` | `180` | How long a lock can go without a heartbeat before another device treats it as abandoned. |
| Lock | `DeviceName` | this machine's hostname | This device's name in the lock file. **Must be unique per device.** |

### Example configuration

`BepInEx/config/cobalt.cloudward.cfg` — created on first launch. Excerpt:

```ini
[Sync]
Enable = false
MountPath =
MarkerFileName = .outward-sync-root
AutoCreateMarker = true
ForkResolution = NewestWins

[Lock]
HeartbeatSeconds = 30
StaleSeconds = 180
# DeviceName defaults to this machine's hostname
DeviceName =
```

## Commands

Write a command into `BepInEx/config/Cloudward_cmd.txt` and it runs on the next poll (results go to the
log).

| Command | What it does |
|---|---|
| `syncstatus` | Current sync state: mode, local/share paths, lock holder, any paused forks. |
| `syncnow` | Force a reconcile and push right now (a pull still needs a fresh launch). |
| `syncresolve <local\|share\|both> [uid]` | Resolve a paused fork — see *If two devices genuinely disagree* above. |
| `synclock` / `synclock release` | Show who holds the lock / release your own after pushing. |
| `syncmarker` | Manually stamp the share-liveness marker into `MountPath`. Optional — it's created automatically unless you set `AutoCreateMarker` to `false`. |
| `selftest` | Run the mod's internal checks and log the results. |

## How it works

The game's own save folder is never touched or redirected — Cloudward treats it as the single working
copy and the mount as a mirror it keeps current. Outward saves are immutable, timestamped snapshot
folders, so syncing is really "copy any snapshot folders the other side doesn't have yet" plus knowing
which side is ahead. That last part is the interesting bit: because Outward only keeps the 15 most
recent snapshots per character, the common history between two devices can eventually get pruned away
entirely, so Cloudward can't always tell a clean catch-up from a real divergence just by comparing
snapshot lists. A small counter on the share, bumped every time a device pushes, resolves that: if
only your local copy has moved since the last time you synced, that's a push; if only the share has,
that's a pull; if both have, that's a genuine fork — resolved automatically per your
`ForkResolution` policy (most-recent-save wins by default), always keeping a backup of the side it
doesn't pick.

## For modders

Cloudward is a standalone gameplay mod, not a library — nothing here is meant to be depended on by
another mod. It uses [ForgeKit](../kits/forgekit.md) for its dev-tooling loop (the command channel and
self-test harness) the same way every mod in this family does.

## See also

- [ForgeKit](../kits/forgekit.md) — the dev-tooling kit behind the command channel
- [Installing](../installing.md) — general setup for this mod family
- [Mods index](./README.md)
