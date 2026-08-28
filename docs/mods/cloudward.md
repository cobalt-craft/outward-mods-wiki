# Cloudward — sync your saves across devices

**Cloudward** is a mod for Outward that keeps a character's save in sync across the machines you play
on — a desktop, a laptop, a Steam Deck — via a mounted or shared folder. Outward has no built-in way
to do this: Steam Cloud is deliberately unsupported for its saves. Optionally it also carries your
**mod folders and mod settings** between those same machines, so a character never arrives on a
device running an older build. Cloudward is for players who play the same character on more than one
device.

**At a glance**
- Type: gameplay mod
- Requires: BepInEx 5 (Mono branch), [ForgeKit](../kits/forgekit.md). No SideLoader.
- Config: `BepInEx/config/cobalt.cloudward.cfg`
- Commands: `BepInEx/config/Cloudward_cmd.txt`
- Ships two files: `BepInEx/plugins/Cloudward/` (the mod) and
  `BepInEx/patchers/Cloudward.Preload.dll` (see *What gets installed*)
- Inert until you turn it on — `[Sync] Enable` is `false` out of the box

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

Back up your `SaveGames/` folder before your first real sync.

**You can see it working.** If the launch pull takes more than a moment — a first sync, or a big
share over a slow connection can run for minutes — a small progress bar and a one-line status appear
at the bottom of the main menu ("checking share…", "syncing saves… 12/44"), ending with a brief
"sync complete — pulled 40" before disappearing. A fast sync shows nothing at all, so on most boots
you'll never see it; there is no setting for it. The `[CLOUDWARD]` lines in `BepInEx/LogOutput.log`
carry the same story in more detail.

### What gets installed

Cloudward is two files, and they go in **different places**:

| File | Goes in | Why |
|---|---|---|
| `Cloudward/` | `BepInEx/plugins/` | The mod itself — the sync loop, the settings, the commands. |
| `Cloudward.Preload.dll` | `BepInEx/patchers/` | Installs a downloaded mod update just before the game loads mods. |

A mod manager or the bundle installer puts both in the right place. **Installing by hand, don't put
`Cloudward.Preload.dll` in `plugins/`** — there it simply does nothing, silently, and mod updates
fetched from the share will never install. It has to be a *patcher* because nothing can replace a mod
DLL or a settings file in a session that is already running: BepInEx loads every mod at startup, so
the swap has to happen before that point.

### Setup

1. Mount (or sync) the same folder on every device — a network share over NFS/SMB, a mapped Windows
   drive, or a local Syncthing folder all work. Pick something with low latency, since the game's quit
   save writes through it.
2. Launch Outward once so the mod writes its config file, then quit. The file is
   `BepInEx/config/cobalt.cloudward.cfg` — **edit it only while the game is closed** (a launch
   rewrites it, so an edit made with the game running is lost).
3. In that file, set two keys under `[Sync]` — `Enable` and `MountPath`:

   ```ini
   [Sync]
   Enable = true
   MountPath = Z:/mnt/shared/outward
   ```

   **Path format:** Outward runs under Proton on Linux/Steam Deck, so `MountPath` is a Wine drive path —
   `Z:` is the filesystem root `/`, so a share mounted at `/mnt/shared/outward` is
   `MountPath = Z:/mnt/shared/outward`. On native **Windows** use a normal path or UNC, e.g.
   `MountPath = S:\OutwardSync` or `MountPath = \\SERVER\OutwardSync`. Every device using the **same
   Steam account** lands in the same save folder on the share automatically.
4. Relaunch, and check the log for whether the share was recognised (below).

**Blessing the share.** Cloudward will not sync into a folder it cannot confirm is really the share.
An unmounted mountpoint looks exactly like an ordinary empty folder, so without that check a sync
could quietly write onto local disk and be lost. The confirmation is a small marker file
(`.outward-sync-root`):

- **Native Windows** (mapped network drive or UNC), or **any share that already holds save data** —
  Cloudward writes the marker itself on first run and there is nothing to do.
- **Linux and Steam Deck with an empty first-run share** — it *can't* be auto-confirmed, because the
  game only sees Wine drive paths and cannot read the system's mount table. Cloudward stays offline
  and says so in the log. Bless it once: write `syncmarker` into
  `BepInEx/config/Cloudward_cmd.txt` while the game runs, or create the marker file by hand. Every
  later launch is automatic.

Set `AutoCreateMarker = false` if you'd rather always bless shares by hand.

To choose how conflicts resolve, set `[Sync] ForkResolution` (e.g. `ForkResolution = ThisDevice`) —
the default `NewestWins` is usually right.

### If two devices genuinely disagree

If the same character was played offline on two devices before either could sync, Cloudward resolves
it automatically per the `[Sync] ForkResolution` setting — and **always backs up the version it
doesn't keep** to `.cloudward-forks/`, so nothing is discarded:

- **`NewestWins`** *(default)* — keep whichever device has the most recent in-game save.
- **`ThisDevice`** — always keep this device's version.
- **`OtherDevice`** — always take the other device's version.
- **`KeepBoth`** — keep this device's version live and preserve the other as a restorable backup.
- **`AskMe`** — don't guess; wait for you to decide with the `syncresolve local|share|both` command
  (`syncstatus` names the paused character).

`syncresolve` is also always available as a manual override under any policy.

### Multiple devices at once

Only one device pushes to the share at a time, tracked by a small heartbeat lock file. If you launch
the game on a second device while a first is actively playing, the second still picks up everything
already on the share — it just holds its own new progress locally until the first device finishes.
The lock also recovers on its own from a crash or a lost connection: a lock that's stopped
heartbeating is treated as abandoned, so a dead session can never permanently block another device.

### Syncing your mods and settings too

The `[Payload]` tier carries your **mod folders** and **mod settings** across the same devices. It is
**off by default and separate from save sync**: turning it on rewrites `BepInEx/plugins`, so it should
be a decision rather than a side effect. Set `[Payload] Enable = true` (it also needs `[Sync] Enable`
and a `MountPath`). The first launch after you enable it only *reports* what it would do; it acts from
the second launch onward.

How it behaves:

- **It only works when something changed.** Each launch it compares a size + modification-time
  signature of the tree; unchanged means nothing is hashed and nothing is copied.
- **Incoming updates apply at the NEXT launch, not this one.** Cloudward downloads the update and
  verifies every file's checksum; the preloader installs it just before mods load next time you start
  the game. If anything fails verification the update is discarded and your install is untouched.
- **Settings sync is an allowlist, not a file copy.** Only the keys named in
  `BepInEx/plugins/Cloudward/overlays/*.cfg.overlay` travel. Everything else in your `.cfg` files —
  including `MountPath`, `DeviceName`, your keybinds, your resolution and your controller
  bindings — is host-local and its bytes are never rewritten.
- **Your saves, logs and per-character mod data never leave** through this tier, whatever else lives
  in those folders.
- **If both devices changed**, it stops and waits for you (`payloadresolve`) instead of guessing — a
  wrong guess would install the wrong code. The replaced version is always backed up first, and
  `payloadrollback` puts it back.
- **Emergency stop without launching the game:** create an empty file named `.cloudward-disable` in
  your Outward game folder. All payload activity halts, including the part that runs at startup.

### Known issues and limitations

- **BepInEx itself is never synced.** The loader (`winhttp.dll`, `BepInEx/core/`, `mono_fix/`) is
  loaded and locked before any mod code runs, so it cannot be updated from a share. Install and
  update BepInEx on each device yourself. Cloudward records a read-only fingerprint of the loader and
  **refuses to adopt a payload published by a device whose loader differs**, rather than installing
  mods that box can't run.
- **A mod update always costs one extra launch.** There is no way to apply it to the session that
  fetched it.
- **`ForkResolution = KeepBoth` preserves the other side as a restorable backup**, not as a second
  character in the load list.
- **`syncnow` pushes only.** Pulling in another device's progress happens at launch, before the
  character list is built; there is no mid-session pull.
- **Settings that are deliberately not portable stay put.** If a key isn't named in an overlay file,
  the payload tier will not move it — by design, not by omission.

## Settings

`BepInEx/config/cobalt.cloudward.cfg`:

| Section | Key | Default | Effect |
|---|---|---|---|
| Sync | `Enable` | `false` | Master switch. Off = vanilla, local-only saves. |
| Sync | `MountPath` | *(empty)* | The shared/mounted folder to sync through. Empty = inert. |
| Sync | `MarkerFileName` | `.outward-sync-root` | Sentinel file that marks the share as actually connected. |
| Sync | `AutoCreateMarker` | `true` | Write the marker automatically when the share can be positively confirmed — it already holds save data, or the platform can see it in the mount table (native Linux, or a Windows network drive / UNC path). An ambiguous empty folder stays offline with a log hint instead, which under Proton is every empty first-run share: bless it once with `syncmarker`. Remembered per mount path, so a later launch with the share disconnected reads as offline and is never re-stamped. `false` = always require `syncmarker`. |
| Sync | `OnMountDown` | `FallbackLocal` | What happens if the share isn't reachable at launch: `FallbackLocal` plays local and warns; `RefuseAndLog` does the same but logs more loudly. |
| Sync | `OnLockHeld` | `FallbackLocal` | What happens to *pushing* while another device holds the lock (pulling always still happens): `FallbackLocal` defers and catches up once it's free; `ForceTake` pushes anyway — only use this if you're sure the other device is gone for good. |
| Sync | `ForkResolution` | `NewestWins` | How a character played offline on two devices is auto-resolved: `NewestWins` = most recent in-game save; `ThisDevice` / `OtherDevice` = always this / the other device; `KeepBoth` = keep this device's live and preserve the other as a backup; `AskMe` = wait for a manual `syncresolve`. The version not kept is always backed up. |
| Lock | `DeviceName` | this machine's name | This device's name in the lock file. **Must be unique per device.** |
| Lock | `HeartbeatSeconds` | `30` | How often the lock's heartbeat (and the share connection) is refreshed. |
| Lock | `StaleSeconds` | `180` | How long a lock can go without a heartbeat before another device treats it as abandoned. |
| Payload | `Enable` | `false` | Master switch for mod + settings sync. Also needs `[Sync] Enable` and a `MountPath`. |
| Payload | `SyncPlugins` | `true` | Sync the `BepInEx/plugins` tree — whole folders, including the data files mods ship beside their DLL. |
| Payload | `SyncConfig` | `true` | Sync the settings keys named in `plugins/Cloudward/overlays/*.cfg.overlay`, and only those. |
| Payload | `Role` | `Both` | `Both` = publish and adopt. `Publisher` = never adopt. `Subscriber` = never write to the share (use on a machine that isn't yours). |
| Payload | `ApplyMode` | `Auto` | `Auto` = a downloaded update installs itself at the next launch. `StageOnly` = download it and wait for `payloadapply`. |
| Payload | `OnConflict` | `Skip` | Both sides changed. `Skip` = touch nothing and wait for `payloadresolve`; `PreferLocal` / `PreferShare` decide automatically at launch. |
| Payload | `PushAtQuit` | `false` | Also publish at quit, catching a mid-session install. Off by default: quitting has a short budget and a plugins tree is large. Publishing normally happens at launch. |
| Payload | `KeepBackups` | `3` | How many pre-replace backups to keep per tier — what `payloadrollback` restores from. |
| Payload | `MaxPayloadMegabytes` | `2048` | Refuse to publish a tier bigger than this; a sudden jump almost always means a mis-set folder. |
| Payload | `ExtraExcludes` | *(empty)* | Extra `;`-separated patterns to keep off the share. Additive only — the built-in exclusions can't be switched off. |

### Example configuration

`BepInEx/config/cobalt.cloudward.cfg` — created on first launch. Excerpt (defaults; set at least
`Enable` and `MountPath` to turn it on):

```ini
[Sync]
## Master switch. Off = vanilla local-only saves.
Enable = false
## Root of the shared/mounted folder to sync through. Empty = inert.
MountPath =
MarkerFileName = .outward-sync-root
AutoCreateMarker = true
## FallbackLocal | RefuseAndLog
OnMountDown = FallbackLocal
## FallbackLocal | ForceTake
OnLockHeld = FallbackLocal
## NewestWins | ThisDevice | OtherDevice | KeepBoth | AskMe
ForkResolution = NewestWins

[Lock]
## Defaults to this machine's name. Must be unique per device.
DeviceName =
HeartbeatSeconds = 30
StaleSeconds = 180

[Payload]
## Mod + settings sync. Requires [Sync] Enable and MountPath.
Enable = false
SyncPlugins = true
SyncConfig = true
## Both | Publisher | Subscriber
Role = Both
## Auto | StageOnly
ApplyMode = Auto
## Skip | PreferLocal | PreferShare
OnConflict = Skip
PushAtQuit = false
KeepBackups = 3
MaxPayloadMegabytes = 2048
ExtraExcludes =
```

Cloudward ships no config-override data tables.

## Commands

Write a command into `BepInEx/config/Cloudward_cmd.txt` and it runs on the next poll, even while the
game is paused (results go to the log). Write `help` to list them all.

| Command | What it does |
|---|---|
| `syncstatus` | Current sync state: mode, local/share paths, lock holder, any paused forks. |
| `syncnow` | Force a reconcile and push right now (a pull still needs a fresh launch). |
| `syncresolve <local\|share\|both> [uid]` | Resolve a paused fork — see *If two devices genuinely disagree* above. |
| `synclock` / `synclock release` | Show who holds the lock / push and release your own. |
| `syncmarker` | Manually stamp the share-liveness marker into `MountPath`. |
| `payloadstatus` | Mod/settings sync state: tiers, generations, what's staged, conflicts, backups, loader fingerprint. |
| `payloadscan [plugins\|config]` | Re-check from cold, ignoring the size + modification-time shortcut. |
| `payloadpush [plugins\|config]` | Publish this device's mods/settings now. |
| `payloadstage [plugins\|config]` | Download the share's version now, ready for the next launch. |
| `payloadresolve <local\|share> [plugins\|config]` | Resolve a mod/settings divergence. |
| `payloadrollback <plugins\|config> [n]` | Restore a tier on the share from a backup (`0` = newest). |
| `payloadapply` | Release an update held by `ApplyMode = StageOnly` so the next launch installs it. |
| `selftest` | Run the mod's internal checks and log the results. |

Cloudward does not add ForgeKit's shared dev-verb pack to this channel — only the commands above,
plus `help` and the scripting verbs (`script`, `scriptstatus`, `scriptcancel`) that every command
channel provides.

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

The one piece worth borrowing is the shape of `Cloudward.Preload.dll`. A BepInEx **preloader patcher**
runs before the chainloader starts, which is the only moment at which `plugins/*.dll` and
`config/*.cfg` are neither open nor locked — so it is the only place a mod can rewrite another mod's
files and have the change take effect. Two traps: the optional teardown method must be named
`Finish()` (not `Finalizer`), and `TargetDLLs` may be empty but a `Patch(ref AssemblyDefinition)`
method must still exist or BepInEx skips the type in silence.

## See also

- [ForgeKit](../kits/forgekit.md) — the dev-tooling kit behind the command channel
- [Installing](../installing.md) — general setup for this mod family
- [Mods index](./README.md)
