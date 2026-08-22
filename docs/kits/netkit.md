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
  protocol version, the map of every registered channel and its version (plus an optional
  per-channel extension string), and the sender's build identity. From that map it works out, per
  channel, which peers can speak it. If a peer's build identity differs from yours, NetKit logs a
  one-time advisory note — a stale install is a common reason co-op misbehaves even when both sides
  read as modded. The handshake also **refreshes itself**: every few seconds NetKit checks whether
  the hello it would send now differs from the one it last sent (a mod's data was retuned
  mid-session, or a channel registered late) and, if so, re-sends it — so peers on either side of
  the session learn about mid-session changes without anyone changing scenes.
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
refused. Even between two NetKit builds, a difference in build identity draws the advisory
mismatch note above — mismatched bundles are the most common source of co-op oddities. This is by
design: the mods ship as one bundle, so every machine in a session is expected to run the same
build.

## Settings

`BepInEx/config/cobalt.netkit.cfg`, section `[Net]`:

| Key | Default | Effect |
|---|---|---|
| `Transport` | `Rpc` | Which Photon backend carries traffic. `Rpc` is the proven relay piggybacked on the game's own network object and is the recommended default. `Event` is an alternative built on Photon's `RaiseEvent` (no game object needed); it is available but off by default — leave it on `Rpc` unless you have a specific reason to switch. |
| `EventCode` | `177` | The Photon event code the `Event` transport uses (0–199; 200+ are reserved). Consulted only when `Transport = Event`. Change only if it collides with another mod's event code. |
| `HeartbeatSeconds` | `30` | While in a room, log one heartbeat line per channel every N seconds (role, peers ready, the consumer's fragment, network time), so a log names when a session went quiet. `0` disables it. |
| `HelloWarnSeconds` | `10` | How long after a peer joins to wait for its hello before warning that it appears unmodded. |
| `HelloRefreshSeconds` | `5` | While in a room, check every N seconds whether the handshake payload changed since the last send (a mid-session data retune, a late-registered channel) and re-send it to peers when it did. `0` disables the refresh — a change then only reaches peers on the next join or scene load. Applied at startup. |
| `VerboseNet` | `true` | Log every send/receive as one line under the owning channel's tag. Turn off to keep only the counters and ring buffer. |
| `DisconnectTimeoutMs` | `0` | Override Photon's disconnect timeout (vanilla is 10000 ms). At the default, a peer silent for longer than 10 s during a long load or a network hitch is torn down and re-read as "gone". Widen to `30000`–`60000` to ride those out. `0` leaves the vanilla value untouched; negative is ignored. Applied once at startup. |

### Example configuration

`BepInEx/config/cobalt.netkit.cfg` — created on first launch. Excerpt:

```ini
[Net]
Transport = Rpc
EventCode = 177
HeartbeatSeconds = 30
```

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

// Or register with a declared role constraint — the guard your handler would otherwise have to
// open with. A violating message is drop-counted (visible in netdump) and the handler is skipped.
ch.Register("mint", HandleMint,
    HandlerRole.RunOnGuestOnly | HandlerRole.SenderMustBeMaster | HandlerRole.SenderMustNotBeSelf);
// Flags: Any (no constraint) · RunOnMasterOnly · RunOnGuestOnly (note: in solo the local player
// IS the master) · SenderMustBeMaster · SenderMustNotBeSelf (drops your own broadcast echo).

// Send. All return bool and never throw; a refused send is counted as a drop.
ch.SendToMaster("hit", payload, extra);   // guest → master
ch.SendToOthers("spawn", payload);        // everyone but me
ch.SendToAll("spawn", payload);           // everyone including me
ch.SendToPlayer(player, "sync", payload); // one actor

// React to peer readiness for THIS channel.
ch.OnPeerReady += info => Flush(info.Actor);   // info: Actor, ChannelVersion, Extension, IsMaster
ch.OnPeerLost  += info => Forget(info.Actor);

// Session events (shared — stop hand-rolling room watches and scene-ready trackers).
Net.OnRoomChanged += c => TearDown(c.OldRoomName, c.NewRoomName);  // join, leave, or room switch;
                                                                   //  null name = "not in a room"
Net.OnSceneReady  += scene => Resync(scene);   // once per (room, scene): loader done + local player
                                               //  resolved. Both roles.
// Each subscriber runs in its own try/catch — your throw is logged, never breaks the others.

// Report a message your own code chose to throw away, so netdump can name it.
ch.CountDrop("spawn", "no-row");

// Session guards live on the static facade.
if (Net.InRoom && Net.IsMaster) { /* … */ }
// Also: Net.Attached, Net.IsGuestInRoom

// ── State mirrors (change-latched outbound sync) ─────────────────────────────
// A StateMirror sends a value only when it CHANGED since the last send that
// actually LANDED — the "latch-on-landed" rule (a failed send retries next
// tick) is enforced by the kit, so you never hand-roll a last-sent field again.
var hp = ch.Mirror("mymod.hp", MirrorTarget.Master,          // or Others / Actor
    build: () => HasPet ? $"{cur}\t{max}" : null,            // null = nothing to mirror
    quantize: s => Round(s));                                // optional compare projection
hp.Tick();               // call on your own cadence; sends iff quantize(build()) changed
hp.Invalidate();         // force the next Tick to re-send (receiver's row was rebuilt)
hp.Reset();              // silently forget (left the room)
// Full knobs via MirrorOptions: Extra (the envelope's extra slot — participates in the
// change key), ResendSeconds (periodic re-send of an UNCHANGED value, loss tolerance),
// and ClearVerb/ClearPayload (a null build after a landed send then owes ONE wire clear,
// retried until it lands). LastPayload/LastExtra expose the last landed bytes — the
// natural source for a flush-to-late-joiner.

// ── Request/Ack (correlated guest → master requests) ─────────────────────────
// For a request that SPENDS something on the answer (an item, a charge): the kit
// mints a correlation token, enforces one-in-flight per verb, times out, and
// drops stale/duplicate acks — exactly one result callback per request.
ch.SendRequest("mymod.spend", payload, timeoutSeconds: 2.5f, r => {
    switch (r.Outcome) {
        case RequestOutcome.Ok:            /* the handler answered: r.Result */ break;
        case RequestOutcome.TimedOut:      /* nothing answered — refuse, spend nothing */ break;
        case RequestOutcome.RefusedNoSend: /* never sent: r.Refusal ("in-flight", "send-failed", …) */ break;
    }
});
// Master side: the kit parses the token BEFORE your handler runs (msg.Payload = your body,
// token stripped) and only reply() can ack — it always carries the real token back to the
// requesting actor on "<verb>.ack". NOT replying is the correct response to an
// unauthorized sender (the requester's timeout answers honestly).
ch.RegisterRequestHandler("mymod.spend", HandlerRole.RunOnMasterOnly, (msg, reply) => {
    if (!Authorized(msg)) return;          // silence — never ack an unbound actor
    reply(DoTheThing(msg) ? "ok" : "refused");
});
// Fire-and-forget verbs should stay plain sends — not every request wants this weight.

// ── ReplicatedRecord stores (announce/refresh/flush/teardown lifecycles) ─────
// For KEYED RECORDS one machine owns and the session must agree on (a guest's
// pet, a broadcast spawn, a cosmetic effigy): the store owns existence,
// identity, freshness, authority and flush — payload bytes and what a row
// MEANS stay yours. Verb strings are configurable; byte-level interop with
// older builds holds only where the consumer already carried the wire key in
// the extra slot (ck.proxy.*/ck.effigy.*) — shipped sk.spawn/sk.gone carry the
// uid in the payload instead, so old/new sk builds do NOT interop (covered by
// the same-bundle rule below).
var store = ch.RegisterStore("proxy", new StoreOptions {
    Authority        = StoreAuthority.Master,   // owners announce → the master binds + holds rows
                                                // (StoreAuthority.Owner = broadcast-to-Others shape:
                                                //  rows live on every OTHER machine, flush targeted)
    ResolveUidOwner  = (uid, slot, actor) =>    // sender↔uid authorization, YOUR game lookup:
        MyResolve(uid, actor),                  //  OwnedBySender / NotOwnedBySender / Unknown
                                                //  (Unknown = DEFER — the re-announce retries)
    RefreshSeconds   = 30f,                     // periodic re-announce of an unchanged record
    FlushOnPeerReady = true,                    // late joiners get the current state: Master
                                                //  authority re-arms the owner latches (the next
                                                //  push re-announces); Owner authority does a
                                                //  TARGETED last-landed-bytes replay to the new
                                                //  peer only. Set false to drive FlushTo(actor)
                                                //  yourself when you must ORDER the replay against
                                                //  sibling traffic (the effigy set-before-stance
                                                //  rule).
    ClearOnOwnerLost = true,                    // OnPeerLost fast path + a paced backstop for
    OwnerMissingSeconds = 45f,                  //  exotic failures (owner replica unresolvable)
    ClearOnRoomChange = true,                   // the room watch lives in the kit, per store
    Verbs = new StoreVerbs { Announce = "mymod.announce", Release = "mymod.release" },
});
// Owner side — push your CURRENT state every tick; the kit latches (fresh /
// changed / periodic; latch-on-landed, so a failed send retries next push):
store.Announce(key, payload, extra);   // key = "ownerUid" or "ownerUid:slot"
store.Release(key, "pet released");    // wire clear retried until it lands
store.Invalidate(key);                 // authority's row torn down out-of-band → re-announce now
// Authority / mirror side — rows + events:
store.OnSet     += (key, payload, meta) => Apply(key, payload);   // meta: SenderActor, IsRebind,
store.OnCleared += (key, reason, meta) => TearDown(key, reason);  //  IsRefresh, Extra
store.TryGet(key, out var row);        // row: Key, OwnerUid, Slot, ActorId, Payload, age stamps
store.RowsSnapshot();
store.FlushTo(actor);                  // Owner authority only: on-demand targeted replay of the
                                       //  last landed bytes (a guest's scene-ready resync request;
                                       //  OnPeerReady is edge-triggered and can't re-fire). Sets
                                       //  are idempotent. Returns the count sent; 0 on Master.
```

**Store authorities — `Master`, `Owner`, `PeerOwned`**

| Authority | Who holds the rows | Receive guard | Owner binding |
|---|---|---|---|
| `Master` | the master, after arbitrating | runs on the master machine only | yes (`ResolveUidOwner`) |
| `Owner` | every machine mirrors the broadcaster | sender must be the master, self-echo dropped | no |
| `PeerOwned` | every machine mirrors every peer | self-echo dropped only | no — refused |

`PeerOwned` is `Owner` without the master-in-the-middle: **any** peer holds and broadcasts its own
rows, and every machine (the master included) mirrors them. Everything `Owner` does in the broadcast
direction — apply-local-first, `SendToOthers`, the targeted flush on a peer's scene-ready edge, the
retried release, `OnSet`/`OnCleared` on every mirror — `PeerOwned` does identically.

*Key from sender.* Dropping the master-sender demand is only safe because a `PeerOwned` receiver
**discards the uid that rode the wire** and recomposes the row key from the message's sender actor
(the slot is preserved). Actor numbers are assigned by the Photon server and stamped by the relay,
so a peer cannot forge one — the only row a sender can ever address is its own. That single
substitution replaces the whole authorization apparatus, which is why an owner-binding resolver is
meaningless here and is **refused at construction** rather than silently ignored. It also means the
table's rebind and foreign-release refusals can never fire over the wire on such a store; they stay
in place and stay unit-tested, but seeing them idle is the design working. Row keys on a `PeerOwned`
store therefore read as `actorId` / `actorId:slot` — read them back with the actor id, not your uid.
Outside a room the local actor is `0`, so `Announce` refuses (no send, no local row) until the join
completes; your next push carries it.

*One logical record per slot.* Because the table key is the **sender**, every consumer key a `PeerOwned` store announces at the same slot lands on the **same row** — two different uids at slot 0 would overwrite each other on every refresh. Use one constant key per store (the warm mirror uses `"warm"`), and the slot when you genuinely need several rows per peer.

*`ReapAbsentActors`* (default `false`) is an opt-in per-store sweep, available to both broadcast
authorities, that drops rows whose bound actor is no longer in the room. `OnPeerLost` is the fast
path, but a peer readiness reset does not raise it, so a departed peer's row can otherwise survive
as a ghost. It is off by default so the shipped spawn / effigy / petfx stores stay byte-identical.
`netdump` shows `authority=`, `reap=on` when the sweep is enabled, and — gated on that same flag —
`present=yes/no` per row, so a store that does not reap does not grow a column nobody acts on.
SpawnKit's warm mirror is the first shipped consumer (`warm` on the `sk` channel: one row per peer,
keyed `"warm"`, reap on).

Registration is the one line:

```csharp
var status = ch.RegisterStore("peerstatus", new StoreOptions {
    Authority         = StoreAuthority.PeerOwned,   // no ResolveUidOwner — it is refused here
    ReapAbsentActors  = true,
    Verbs = new StoreVerbs { Announce = "mymod.status", Release = "mymod.status.clear" },
});
```

Pick `PeerOwned` when each peer is simply the truth about its own state and nothing needs to
arbitrate (per-peer presence/status fan-out). Pick `Owner` when the master mediates the broadcast
(the shipped effigy/spawn shape). Pick `Master` when the master must arbitrate the rows themselves
and a sender↔uid authorization is required.

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
  readiness doesn't change. The hello refresh (see above) re-sends automatically when your own
  extension changes — but keep the extension callback **deterministic** for a given data state: a
  value that changes on every call would make the refresh re-send every interval.
- **Late channel registration works.** A channel registered after a peer's hello already arrived
  (e.g. at pack-load time) is immediately evaluated against the stored hellos, so compatible peers
  read as ready without waiting for a re-hello.
- **`RefreshRpcMonoBehaviourCache()` is a harmless no-op precaution, NOT mandatory.** The
  shipped game's `PhotonNetwork.UseRpcMonoBehaviourCache` is `false`, so PUN re-scans the view's
  MonoBehaviours on every RPC dispatch and a component added after the view existed is found
  without it. NetKit still calls it after mounting `NK_Bus` because it costs nothing; an older
  SpawnNet header claimed the call was mandatory, and that claim was wrong (decompile-verified at
  the 2026-07-18 extraction — `docs/netkit-plan.md`).
- **Room-join asymmetry.** A room join clears the ready set without firing `OnPeerLost`. For
  room-change cleanup, subscribe to `Net.OnRoomChanged` rather than relying on ready/lost symmetry.
- **Mirror null-build semantics.** A `build` that returns `null` means "nothing to mirror right
  now" and leaves the latch untouched (a transient window — say the player object is mid-reload —
  must not eat a pending change). Only a mirror configured with a `ClearVerb` treats null as "the
  state went away": it sends one clear and forgets. `""` is a real payload, distinct from null.
- **Request results are answers, not verdicts.** The kit correlates; it never interprets. An
  `Ok(result)` means the handler replied — whether `result` authorizes your spend is your contract
  (compare against your own constants; treat anything unknown as a refusal).
- **Store keys are entity ids.** A store key is `ownerUid:slot`; a bare key is slot 0 in BOTH
  directions — an old peer's bare announce lands at `uid:0`, and a new peer emits the bare form for
  slot 0, so the shipped ownerUid-keyed wire keeps interoperating while multi-record consumers
  (a second pet, a hireling) get slots. Slots > 0 (and non-empty consumer `extra` on `Announce`)
  are NEW capabilities: an old peer reads them as unknown bare keys — new features degrade, shipped
  ones never break.
- **Owner-authority local apply is unconditional.** An Owner-authority `Announce` feeds the local
  row table (raising `OnSet` here) BEFORE the broadcast and regardless of the send outcome, so the
  announcing machine's own view never lags a transport outage; the latch still advances only on a
  landed send, so the wire side retries. Live consumers: CompanionKit's `effigy` store (Owner
  authority) and its `proxy` store (Master authority — the guest-pet
  proxy lifecycle), both on the `ck` channel with wire bytes identical
  to the pre-store builds; and SpawnKit's `spawn` store (Owner authority)
  on the `sk` channel — see the cutover note below. NB a Master-authority consumer's OWNER side
  may keep hand-sending the
  same bytes instead of using the store's `Announce` book (the proxy's guest half does — its
  cadence is entangled with consumer policy); the store's receive half neither knows nor cares.
- **⚠ `sk` WIRE CUTOVER at SpawnKit 0.4.0 — old and new builds do NOT interop.**
  Migrating a consumer onto the store is byte-transparent only if it ALREADY carried its wire key
  in the message `extra` slot, which `ck.proxy.*`/`ck.effigy.*` did. `sk.spawn`/`sk.gone` did not:
  the shipped build sent `extra=""` with the spawn uid inside the payload. The payload bytes are
  unchanged and the uid still rides inside them, but the KEY half moved, so a pre-0.4.0 announce
  decodes an empty wire key at a 0.4.0 receiver and drops `empty-key`, and a pre-0.4.0 `sk.gone`
  drops `no-row`. Channel compatibility is exact ordinal version equality, so the remedy is the
  version: SpawnKit went `0.3.1` → `0.4.0` and a mixed pair now refuses LOUDLY at the handshake
  rather than connecting and silently dropping every spawn. Payload-format compatibility is NOT
  interop — do not read it as such. **The general rule this instantiates: if a consumer's shipped
  messages did not already carry the wire key in `extra`, migrating it to a store is a breaking
  wire change, and the channel version MUST be bumped in the same commit.** Both boxes on the same
  bundle is the standing NetKit rule.
- **The store owns the lifecycle, not the meaning.** Sender↔owner authorization (defer on an
  unresolvable uid, refuse a non-owner), the rebind rule (a new actor may claim a key only after
  the bound actor actually left — a reconnect, never a live hijack), release authorization (only
  the bound actor may release), the owner-missing backstop and the room-change teardown are all the
  kit's. What a row *does* (spawn an anchor, build a body) belongs in your `OnSet`/`OnCleared`.
  Refusals surface as uniform drop reasons in netdump: `owner-unresolvable`, `not-owner`,
  `rebind-refused`, `no-row`, `wrong-sender`, `empty-key`.
- **The compute half is unit-tested.** The hello codec, channel-version compatibility, peer-ledger
  timing, counters, heartbeat formatter, the mirror latch, the request correlation/timeout book,
  and the record store's key codec / row state machine / announce latch live in `NetKit.Core` with
  no game references.

## See also

- [Kits index](./README.md)
- [ForgeKit](./forgekit.md) — the dev-tooling kit NetKit builds on (command channel, self-test)
- [CompanionKit](./companionkit.md) — consumer, channel `ck`
- [SpawnKit](./spawnkit.md) — consumer, channel `sk`
- [Wiki home](../README.md)
