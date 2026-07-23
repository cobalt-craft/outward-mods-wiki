# NetKit — shared Photon co-op transport for mods

**NetKit** is a kit for Outward that gives mods one shared way to send and receive their own state
across a Photon co-op session. Instead of every mod standing up its own network relay, they all
share NetKit's relay, handshake, and diagnostics. It's a library — players never interact with it
directly.

**At a glance**
- Type: reusable library (kit)
- Requires: BepInEx 5 (Mono branch), [ForgeKit](./forgekit.md)
- Config: `BepInEx/config/cobalt.netkit.cfg`
- Commands: `BepInEx/config/NetKit_cmd.txt`
- No SideLoader dependency

## For players

You won't see NetKit in-game — it arrives as a dependency of the mods you install and quietly
carries their co-op traffic. One thing matters to you: **everyone in a co-op session needs the same
mod build.** A player running an older, pre-NetKit build reads as unmodded to the others, and the
modded features stay inert toward them. Because these mods ship together as one bundle, keeping
every player on the same bundle keeps co-op working.

## How it works

A single relay carries every mod's messages. Each message rides one shared wire envelope
`(channel, verb, sequence, extra, payload)`, and the relay routes it by **channel → verb**. A mod
registers a **channel** (an id plus a version), registers **verbs** on it, and sends and receives
on that channel without ever touching Photon itself. [CompanionKit](./companionkit.md) uses channel
`ck`; [SpawnKit](./spawnkit.md) uses channel `sk`.

What NetKit owns so a consumer mod doesn't have to:

- **One handshake.** When peers join or a scene loads, NetKit sends a single `nk.hello` carrying the
  protocol version and the map of every registered channel and its version (plus an optional
  per-channel extension string). From that map it works out, per channel, which peers can speak it.
- **A peer ledger and absence detector.** Peer readiness is surfaced per channel
  (`OnPeerReady` / `OnPeerLost` / `IsPeerReady` / `ReadyCount`) — a peer can support one channel but
  not another. If a joined peer never sends a hello within a grace window, NetKit warns once that it
  appears unmodded and treats every co-op feature as inert toward it.
- **Diagnostics.** Per-channel message counters, a ring-buffer trace of recent traffic, and an
  optional heartbeat line (all timestamped against Photon's clock), plus a watcher for Photon's own
  "unknown PhotonView" warnings. Logged under each channel's own tag so a mod's traffic stays easy
  to grep.

### Compatibility caveat

New builds speak only NetKit's wire format. A pre-NetKit peer (an older build that predates this
layer) doesn't answer the hello, so the handshake reads it as unmodded and co-op cooperation is
refused. This is by design — the mods ship as one bundle, so every machine in a session is expected
to run the same build.

## Settings

`BepInEx/config/cobalt.netkit.cfg`, section `[Net]`:

| Key | Default | Effect |
|---|---|---|
| `Transport` | `Rpc` | Which Photon backend carries traffic. `Rpc` is the live-proven relay piggybacked on the game's own network object. `Event` is an alternative built on Photon's `RaiseEvent` (no game object needed); it is implemented but opt-in until it clears live testing. |
| `EventCode` | `177` | The Photon event code the `Event` transport uses (0–199; 200+ are reserved). Consulted only when `Transport = Event`. Change only if it collides with another mod's event code. |
| `HeartbeatSeconds` | `30` | While in a room, log one heartbeat line per channel every N seconds (role, peers ready, the consumer's fragment, network time), so a log names when a session went quiet. `0` disables it. |
| `HelloWarnSeconds` | `10` | How long after a peer joins to wait for its hello before warning that it appears unmodded. |
| `VerboseNet` | `true` | Log every send/receive as one line under the owning channel's tag. Turn off to keep only the counters and ring buffer. |
| `DisconnectTimeoutMs` | `0` | Override Photon's disconnect timeout (vanilla is 10000 ms). At the default, a peer silent for longer than 10 s during a long load or a network hitch is torn down and re-read as "gone". Widen to `30000`–`60000` to ride those out. `0` leaves the vanilla value untouched; negative is ignored. Applied once at startup. |

## Commands

Write a verb into `BepInEx/config/NetKit_cmd.txt` and it runs on the next poll (even while the game
is paused). `help` or an unknown verb lists them all.

| Verb | Does |
|---|---|
| `netdump` | Co-op census on this machine: transport and attach state, per-channel verbs/peers/counters/ring buffer, the hello ledger, and the Photon-view diagnostics. |
| `selftest` | Runs the self-test (`[SELFTEST] PASS/FAIL … DONE`): hello codec round-trip, compatibility calc, peer-ledger timing, counters, and — when in a room — a live transport loopback. |
| `netmute [on\|off\|status]` | `on` suppresses this box's outgoing hellos so it reads as unmodded to peers; a staging tool for testing incompatible-peer behavior. Incoming handling is unchanged. Session-only, default off; bare `netmute` reports status. |

## For modders

Depend on NetKit like any kit — a `Private=false` reference plus
`[BepInDependency("cobalt.netkit")]` — then register a channel in your plugin's `Awake` and use it.

```csharp
using NetKit;

// Register a channel: an id + a version string. Returns a NetChannel.
var ch = Net.RegisterChannel("mymod", "1.0.0", new ChannelOptions {
    LogTag            = "MYNET",                       // traffic/heartbeat tag (default: id upper-cased)
    HelloExtension    = () => MyStateHash(),           // opaque string folded into every hello (optional)
    HeartbeatFragment = () => $"widgets={_widgets}",   // text appended to the heartbeat line (optional)
});

// Register verb handlers (boot-time; last registration for a verb wins).
ch.Register("spawn", msg => {
    // msg: Verb, Payload, Extra, SenderActor, SenderIsMaster, SenderIsSelf
    Apply(msg.Payload);
});

// Send. All return bool and never throw; a refused send is counted as a drop.
ch.SendToMaster("hit", payload, extra);   // guest → master
ch.SendToOthers("spawn", payload);        // everyone but me
ch.SendToAll("spawn", payload);           // everyone including me
ch.SendToPlayer(player, "sync", payload); // one actor

// React to peer readiness for THIS channel.
ch.OnPeerReady += info => Flush(info.Actor);   // info: Actor, ChannelVersion, Extension, IsMaster
ch.OnPeerLost  += info => Forget(info.Actor);

// Report a message your own code chose to throw away, so netdump can name it.
ch.CountDrop("spawn", "no-row");

// Session guards live on the static facade.
if (Net.InRoom && Net.IsMaster) { /* … */ }
// Also: Net.Attached, Net.IsGuestInRoom
```

**API notes and traps**

- **Send targets and solo play.** In Outward's offline session the local player is the master.
  `SendToAll` and `SendToMaster` self-execute locally even offline; `SendToOthers` silently no-ops
  in solo. So a master → `SendToOthers` broadcast must always be paired with a local call that
  applies the effect on this machine first (receivers drop the self-echo). Prefer `SendToAll` for a
  genuine "everyone including me" effect. Gate every guest → master send on `Net.IsGuestInRoom` with
  the local effect in the else-branch, or you'll double-apply in solo.
- **Unknown verbs and errors are contained.** A verb that isn't registered on a channel is dropped
  with a warn-once and a counter (forward-compatible with newer senders). Handlers run inside a
  try/catch with a per-verb error counter — one mod's throw can't kill the relay.
- **Extension strings drift.** If a peer re-sends its hello with a changed extension for your channel
  (e.g. a live data retune), `OnPeerExtensionChanged` fires instead of re-raising `OnPeerReady`;
  readiness doesn't change.
- **Room-join asymmetry.** A room join clears the ready set without firing `OnPeerLost`. If you need
  cleanup on a room change, watch room identity yourself rather than relying on ready/lost symmetry.
- **The compute half is unit-tested.** The hello codec, channel-version compatibility, peer-ledger
  timing, counters, and heartbeat formatter live in `NetKit.Core` with no game references.

## See also

- [Kits index](./README.md)
- [ForgeKit](./forgekit.md) — the dev-tooling kit NetKit builds on (command channel, self-test)
- [CompanionKit](./companionkit.md) — consumer, channel `ck`
- [SpawnKit](./spawnkit.md) — consumer, channel `sk`
- [Wiki home](../README.md)
