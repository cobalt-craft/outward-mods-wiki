# Kit versioning and compatibility

Every mod in this family is built on shared **kits** (ForgeKit, NetKit, CompanionKit, …). Each kit
is its own BepInEx plugin and its own Thunderstore package — a mod never bundles a copy of a kit.
That is the standard shape for shared libraries in the BepInEx world (R2API, SideLoader, Jotunn all
work this way) and it means there is exactly **one copy of each kit on disk**, which every mod binds
to at runtime.

This page is the contract that keeps that arrangement safe, and what to do when it is not.

## The rule for players and testers

**Install everything from one release.** Either let a mod manager resolve the Thunderstore pins, or
— for a private alpha bundle — install **one bundle that contains every mod you run**. Never layer
two alpha bundles cut on different days: the second one overwrites the kits the first one was built
against, and the older mod then fails at its first call, not at load.

If that happens you will see it in two places:

- an on-screen notice: *"Mod version mismatch … Reinstall all mods from ONE bundle"*;
- `BepInEx/LogOutput.log` lines tagged `[CONTRACT]` (per mod, what it was built against vs. what is
  running) and `[STAMP]` (every mod's build stamp, grouped — two groups means two builds are mixed).

The alpha installer also refuses to overwrite a bundle's kits with a different bundle's kits unless
you confirm. The fix is always the same: ask for (or cut) a single bundle containing all the mods,
and install that.

## How it works

BepInEx 5 resolves duplicate plugin GUIDs by loading the **highest-versioned** copy, and binds every
assembly reference to the highest copy present. So "newest kit wins" is the runtime rule, and it is
only safe when the newer kit is **binary-backward-compatible** — additive changes only. Removing or
re-signing a public member breaks every consumer compiled against the old shape with a
`MissingMethodException` / `MissingFieldException` at first use. Three mechanisms make that visible
instead of mysterious:

### 1. `[BepInDependency(Kit.Plugin.GUID, Kit.Plugin.VERSION)]` — the floor BepInEx enforces

Every consumer declares each kit with the kit's `VERSION` as the second argument. In BepInEx 5 that
is `MinimumVersion`: a plain `x.y.z` means *at least*. Because `Kit.Plugin.VERSION` is a `const`,
the C# compiler inlines the value at the **consumer's** build — the floor is always "the version I
was compiled against", with nothing to regenerate. A consumer whose kit is too old is refused by the
chainloader before its `Awake` ever runs, logged as
`Could not load [Consumer] because it has missing dependencies: cobalt.kit (x.y.z)`.

### 2. `ForgeKit.KitContract` — the boot-time handshake

Each consumer also calls, literally, once per kit:

```csharp
ForgeKit.KitContract.Declare(NAME, NetKit.Plugin.GUID, NetKit.Plugin.VERSION);
```

Same trick — the inlined `VERSION` is the built-against version. On the first frame (after every
plugin's `Awake`) ForgeKit compares each declaration with the running kit's `VERSION` and its
**`COMPAT_SINCE`** floor and logs one `[CONTRACT]` line per pair:

| built against | vs running | verdict |
|---|---|---|
| equal | — | `match` |
| older, ≥ `COMPAT_SINCE` | newer additive kit | `newer kit, binary-compatible` |
| older, < `COMPAT_SINCE` | kit removed surface since | **`VERSION SKEW`** (error + toast) |
| newer | stale kit | **`VERSION SKEW (consumer AHEAD)`** (error + toast) |

`COMPAT_SINCE` is a `public const string` on every kit with consumers: the oldest kit version a
consumer may have been compiled against and still bind. The release train moves it forward
automatically whenever its ABI gate sees a public member removed, re-signed, or added to a public
interface; additive releases leave it alone.

### 3. `[STAMP]` — the zero-discipline backstop

Every assembly in the workspace is stamped with the git commit it was built from. ForgeKit groups
all `cobalt.*` plugins by stamp at boot. One group is normal. Two groups is the alpha-channel
failure in one line — whether or not anyone bumped a version. On an alpha-bundle install (the
installer leaves `BepInEx/cobalt-bundles/*.json`) this also raises the on-screen notice; on a
Thunderstore install it is log-only, because kits from different release trains are normal there.

## The rule for kit authors

- **Additive only within a major version.** New methods, new overloads, new types: fine. Never
  remove or re-sign a public member in a minor/patch release. An optional parameter added to an
  existing method *is* a re-signing (the old arity vanishes from the IL) — add an overload instead.
- **Adding an abstract member to a shipped public interface is a BREAK, not an addition** —
  everyone who *implements* that interface was compiled against the shorter member list, so give
  the new member a `virtual` default on the shared base class and expect the floor to move (this is
  what carried `CompanionKit` `COMPAT_SINCE` to `0.4.10`, where `ICompanionSettings` first grew).
- **A genuine break moves `COMPAT_SINCE`** — the release train does this for you when it detects
  removed surface, and it refuses to publish a kit alone when an in-plan consumer would be stranded.
- **`scripts/check-versions.sh` keeps FOUR hand-written version numbers equal**: `manifest.json`
  (the source of truth — `package-release.sh` names the zip from it), `thunderstore.toml`
  `versionNumber`, `Plugin.cs` `VERSION`, and the csproj `<Version>`. A fifth copy exists but is NOT
  checked because it cannot drift: the matching `core/<Kit>.Core` assembly derives its version
  automatically from the kit's in `Directory.Build.targets`. Assembly
  identity (`AssemblyVersion`) is frozen at `MAJOR.0.0.0`; the real number lives in
  `FileVersion`/`InformationalVersion`, the `[BepInPlugin]` attribute and the package.
- `Kits.Integration.Tests` asserts every `[BepInDependency]` carries its `VERSION` floor and has its
  matching `KitContract.Declare`, and that every consumed kit declares a parseable `COMPAT_SINCE`.

## Where things live

| What | Where |
|---|---|
| Verdict table (pure, unit-tested) | `src/ForgeKit/KitContractRules.cs`, `tests/ForgeKit.Tests/KitContractRulesTests.cs` |
| Boot-time report + stamp census + toast | `src/ForgeKit/KitContract.cs`, `src/ForgeKit/Plugin.cs` |
| Call-site parity gate | `tests/Kits.Integration.Tests/KitContractParityTests.cs` |
| Version policy (core inherits, `AssemblyVersion` frozen) | `Directory.Build.targets` |
| Release train ABI gate + `COMPAT_SINCE` bump | `scripts/release-train.py` (`abi_break_signatures` = `removed_signatures` + `added_interface_signatures`, `read_compat_since`) |
| Alpha bundles: one tester, one bundle | `scripts/release-alpha.sh --profile`, `packaging/profiles/`, `packaging/install.ps1` |

## Why not …

- **Bundle a private copy of each kit into each mod (ILRepack)?** Kits apply Harmony patches; N
  private copies apply them N times. Also duplicate-type identity across mods. Rejected.
- **Strong naming / binding redirects?** Inert under Mono + BepInEx — resolution is by simple name
  and BepInEx's own resolver picks the highest copy. No effect.
- **Split every kit into smaller packages (R2API-style)?** The kits are already split by concern;
  that lever is available if one kit's blast radius ever grows too large.
