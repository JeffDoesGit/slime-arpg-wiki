# Steam identity (partial)

*Built in roadmap item S-0.3 (contract C-7), provisional on [`docs/decisions/DS-0.4.md`](../decisions/DS-0.4.md) §5.3–§5.5 until the real App ID exists and Jeff records the Steamworks decision in the GDD. Design rule: GDD 7.8 (multiplayer saves keyed to the validated Steam ID).*

## What it is

GDD 7.8 keys every multiplayer character record to a **validated** Steam ID — one the engine's own
ticket check has confirmed, not one a client merely claims. This row wires that check in, behind a
switch that is off for everyone who has not turned it on, and gives gameplay code one place to read
the result: `FSteamIdentity` (`Source/SandboxARPG/Server/SteamIdentity.h`).

It is deliberately partial. There is no real App ID yet — that needs a Steamworks partner account
and the Steam Direct fee, both outside the repository (`DS-0.4` §3) — so this row runs against
Valve's public test app (480) and stays off by default. Contract **C-7 stays open**: proving the
check runs is not the same as proving it refuses a forged or expired ticket, and one developer's
own Steam account cannot show that (`DS-0.4` §5.4).

## The three switches

Three things must all be on for a session-ticket to be checked at all, and they live in three
different places on purpose:

1. **`OnlineSubsystemSteam`'s own `bEnabled`**, in `Config/DefaultEngine.ini`. Tracked, and kept
   `false` for everyone on `main`. A developer who wants to run the real check turns it on in their
   own user-layer `Engine.ini` only — never in the tracked file (`CLAUDE.md` 7.13). This is the
   engine's switch: with it off, the active online subsystem is `OnlineSubsystemNull`, as it always
   was, and nothing below this line runs any differently.
2. **`USandboxServerRules::Get().bSteamAuth`**, this project's own server rule (default `false`).
   With the engine switch on but this one off, Steam is technically active but this project's own
   code — the `PostLogin` log line and `FSteamIdentity::IsValidated()` — stays quiet, as if it were
   off.
3. **`[PacketHandlerComponents]` `Components=OnlineSubsystemSteam.SteamAuthComponentModuleInterface`**,
   in `Engine.ini` — the module prefix and the dot are load-bearing, not decoration. This is the
   engine's own switch for the ticket handshake itself (`OnlineAuthInterfaceSteam.cpp:43-58`) —
   without this line `FOnlineAuthSteam` never enables and no ticket is ever fetched or checked,
   whatever the other two switches say. `PacketHandler::AddHandler(const FString&)`
   (`PacketHandler.cpp:398-503`) treats a `Components` entry with no `.` as a loadable module name
   (`FModuleManager::LoadModulePtr`) — a path `OnlineSubsystemSteam` never registers for its auth
   component, so a bare `SteamAuthComponentModuleInterface` line registers no handler at all
   (`PacketHandlerLog: Warning: Unable to Load Module: SteamAuthComponentModuleInterface`), even
   though it is enough to satisfy `FOnlineAuthSteam`'s own looser substring test. Only an entry
   containing a `.` is resolved through `StaticLoadClass` against the
   `USteamAuthComponentModuleInterface` `UHandlerComponentFactory` (`OnlineAuthHandlerSteam.h:63-70`)
   — the same shape the engine's own `Engine.EngineHandlerComponentFactory(StatelessConnectHandlerComponent)`
   entry uses (`NetDriver.cpp:2402`). Every participant, host and joiner alike, needs the same line
   in their own user-layer `Engine.ini`: a host with it and a joiner without never completes the
   handshake (the server sits in `WaitingForKey`, resending every 2 s). `FSteamIdentity::IsValidated()`
   now requires all three, and repeats this same `StaticLoadClass` resolution rather than the bare
   substring test, so a misconfigured bare-name line reads as not validated.

All three must be on for `AMonsterGameMode::PostLogin` to log an identity, and Play In Editor cannot
exercise any of this: `FOnlineSubsystemSteam::IsEnabled` requires `-game` or a dedicated server
(`OnlineSubsystemSteam.cpp:485-489`), so only a `-game` launch or a packaged build can turn Steam on.

See [`../server-config.md`](../server-config.md) for the exact ini blocks and where they go.

## What actually validates the ticket

Nothing in this project's own code fetches or checks a ticket. The engine's `FOnlineAuthSteam`
(`OnlineAuthInterfaceSteam.cpp`, verified against UE 5.8 source) does that entirely inside the
packet handler, before a connection is ever accepted:

- It turns itself on when `SteamAuthComponentModuleInterface` is listed under
  `[PacketHandlerComponents]` in a developer's own `Engine.ini`.
- The joining client fetches a ticket with `ISteamUser::GetAuthSessionTicket`.
- The host validates it with `ISteamGameServer::BeginAuthSession` (a dedicated server) or
  `ISteamUser::BeginAuthSession` (a listen host — this project's only tier today, `DS-2.2`).
  `k_EAuthSessionResponseOK` is success; a refused ticket disconnects with "Host closed the
  connection." — an engine string, not a project one.

`bUseSteamNetworking` stays `false` (`Config/DefaultEngine.ini`), so the IP driver carries every
packet exactly as before; the auth handler validates over that driver rather than replacing it
(`DS-0.4` §5.3).

## The one accessor (contract C-7)

```cpp
FUniqueNetIdRepl FSteamIdentity::GetValidatedNetId(const APlayerController* PlayerController);
bool FSteamIdentity::IsValidated();
```

`GetValidatedNetId` reads `APlayerController::PlayerState::GetUniqueId()` — the engine's own login
flow already sets a player state's unique net id from the connection before any project code runs,
so this is a read, not a second check. It returns an invalid id for a null controller or a
not-yet-logged-in one; it never asserts.

`IsValidated()` is `true` only when `bSteamAuth` is on, the process's active online subsystem is
actually Steam (`IOnlineSubsystem::Get()->GetSubsystemName() == STEAM_SUBSYSTEM`), **and**
`[PacketHandlerComponents]` names an entry that actually resolves to the Steam auth handler
factory — `PacketHandler::AddHandler`'s own `StaticLoadClass` test against
`UHandlerComponentFactory` (`PacketHandler.cpp:398-503`), not `FOnlineAuthSteam`'s looser substring
test (`OnlineAuthInterfaceSteam.cpp:43-58`), which a bare, non-dotted line also satisfies without
registering a handler. Calling code should treat an id from `GetValidatedNetId` as
a real Steam ID only when `IsValidated()` also says so — with any of the three off, whatever
`GetUniqueId()` returns belongs to `OnlineSubsystemNull`, not a checked Steam ticket.

This is the shape a later persistence system (`DS-2.1`, contract C-4) will key the character record
from; nothing here writes a record (`CLAUDE.md` 1.9 — provisional code never persists).

## Where the log lines are

- `AMonsterGameMode::PostLogin` — while `bSteamAuth` is on, logs
  `MonsterGameMode: <controller> logged in; Steam id <id>, ticket validation pending (the engine
  kicks this connection if it fails)`, or a warning naming which half failed (`bSteamAuth` on but
  `IsValidated()` false, or no id) otherwise. This is never a validated login: the engine readies
  the connection and runs `PostLogin` before its asynchronous ticket check resolves
  (`OnlineAuthHandlerSteam.cpp:330-368`), and kicks it a tick later on failure
  (`FOnlineAuthSteam::Tick`, `OnlineAuthHandlerSteam.cpp:410-437`). Binding the auth-result delegate
  to know the real per-connection outcome here is owed, alongside the two-process proof below.
- `Slime.ShowSteamIdentity` (compiled out of Shipping) — prints `bSteamAuth`, whether Steam is the
  active subsystem, and every connected player controller's id and validated state, on demand.

## Verified against the running engine (2026-09-23, and the live join check below)

With the Steam client running and both switches on in a local user-layer config, a `-game` process
on this project logged Steam's own SDK initializing successfully — `LogSteamShared: Steam SDK
Loaded!`, `LogOnline: STEAM: [AppId: 480] Client API initialized 1`, `OSS: Created online subsystem
instance for: Steam` — confirming the plugin and config wiring in this row.

### The owed two-process proof, run against real LFS content (2026-09-23)

The worktree used for the first check had unsmudged Git LFS content (`GIT_LFS_SKIP_SMUDGE=1`), so
no map could load. Smudging it (`git lfs pull`) and rerunning both parts:

**Tracked defaults, no `-nosteam`.** A host and a joiner (`UnrealEditor.exe <uproject>
/Game/Levels/Lvl_Slice?listen -game` and `... 127.0.0.1 -game`) both logged `OSS: Created online
subsystem instance for: NULL` and `ServerRules: steam — bSteamAuth=0`; the joiner logged `Welcomed
by server`. Nothing Steam-specific initialises and the join works exactly as before this row — the
"inert for everyone" claim now rests on a run, not only on reading the engine source.

**Steam on** (both switches plus the `[PacketHandlerComponents]` line, all three in the worktree's
own `Saved/Config/WindowsEditor/Engine.ini` and `Game.ini`, never the tracked files, removed after
the check). The host logged the Steam SDK and OSS lines above, `ServerRules: steam — bSteamAuth=1`,
then its own local controller's `PostLogin` line, exactly as the review's fix reworded it:

```
MonsterGameMode: BP_TopDownController_C_0 logged in; Steam id 76561198078624306,
ticket validation pending (the engine kicks this connection if it fails)
```

The second process's join then failed — **not** from Steam refusing a duplicate session on one
account, which is what this check expected to find. The host log instead shows:

```
PacketHandlerLog: Warning: Unable to Load Module: SteamAuthComponentModuleInterface
```

`PacketHandler::AddHandler(FString)` (`Engine/Source/Runtime/PacketHandlers/PacketHandler/Private/PacketHandler.cpp:474`)
resolves a `[PacketHandlerComponents]` entry through
`FModuleManager::LoadModulePtr<FPacketHandlerComponentModuleInterface>` — it needs an
`IMPLEMENT_MODULE`-registered module of that exact name. `OnlineSubsystemSteam` never registers
one; its only `IMPLEMENT_MODULE` is `FOnlineSubsystemSteamModule` itself
(`OnlineSubsystemModuleSteam.cpp:8`). `USteamAuthComponentModuleInterface`
(`OnlineAuthHandlerSteam.h:64`) is a `UHandlerComponentFactory`, a `UObject` subclass reached a
different way — not a loadable module. So the exact ini line this page's "three switches" section
and `IsValidated()` both rely on can never register the real handshake component. `FOnlineAuthSteam`'s
own constructor (`OnlineAuthInterfaceSteam.cpp:43-58`, the lines `IsValidated()` was written to
match) only string-matches the same ini entry to set its own `bEnabled` — it never checks that the
module actually loaded — so no ticket was ever fetched or checked on either process. The joining
connection reached the server carrying a `Null`-typed unique id (the machine-name id
`OnlineSubsystemNull` always issues) while the server's active platform was Steam, and stock engine
code refused it: `AGameModeBase::PreLogin` (`GameModeBase.cpp:684-698`) computes
`bUniqueIdCheckOk = (!UniqueId.IsValid() || UOnlineEngineInterface::Get()->IsCompatibleUniqueNetId(UniqueId))`,
false here, so `PreLogin failure: incompatible_unique_net_id` on the host and `NMT_Failure
incompatible_unique_net_id` / `Connection failed; returning to Entry` on the joiner.

A further defect the same run surfaced: `Slime.ShowSteamIdentity` on the host printed
`BP_TopDownController_C_0: id=76561198078624306 validated=1` for the very login whose handshake
component had failed to load. `IsValidated()`'s `[PacketHandlerComponents]` check only confirms the
ini *line* is present, not that the module actually loaded, so it reports "validated" for a
connection that underwent no ticket check at all.

**The correct registration path, found 2026-09-23.** `PacketHandler::AddHandler(const FString&)`
(`PacketHandler.cpp:398-503`) does not only resolve a `Components` entry as a loadable module: an
entry containing a `.` (line 443) is instead resolved via `StaticLoadClass(UHandlerComponentFactory::StaticClass(),
nullptr, *ComponentName)` against a `UHandlerComponentFactory` — exactly what
`USteamAuthComponentModuleInterface` is (`OnlineAuthHandlerSteam.h:63-70`) — and only the bare,
non-dotted form falls through to the module-name lookup this page originally documented. The
engine's own code proves the shape: `UNetConnection::InitHandler` (`NetConnection.cpp:762`) adds its
stateless-handshake component as
`"Engine.EngineHandlerComponentFactory(StatelessConnectHandlerComponent)"`, and `NetDriver.cpp:2402`
uses the identical pattern. The correct line is therefore
`+Components=OnlineSubsystemSteam.SteamAuthComponentModuleInterface` — the module name, a `.`, then
the factory class — which is what `docs/server-config.md`, `docs/decisions/DS-0.4.md` §5.2 and the
commented example in `Config/DefaultEngine.ini` already state; this page's "three switches" section
and the accessor description above have been corrected to match. `FSteamIdentity::IsValidated()`
(`SteamIdentity.cpp`) now performs this same `StaticLoadClass` resolution instead of the earlier
substring test, so a bare, unregistering line reads as not validated rather than as validated.

**Re-run the same evening did not reproduce a clean validated join, for two reasons unrelated to
the ini line.** First: the worktree's `Saved/Config/WindowsEditor/Engine.ini` — the only place these
switches may live (7.13) — was found completely deleted a few seconds after the host process
launched, with no client connected yet, even though `bSteamAuth=true` in the sibling `Game.ini`
(backed by a `UPROPERTY(Config)` on `USandboxServerRules`) survived. The `[OnlineSubsystem]`,
`[OnlineSubsystemSteam]` and `[PacketHandlerComponents]` sections are plain, non-reflected keys with
no `UPROPERTY` behind them; something in this engine's config layer (the file's own
`;METADATA=(Diff=true, UseCommands=true)` header names a newer, diff-tracked save format) rewrites
or prunes this exact file early in the session down to whatever it tracks as a live runtime diff,
which does not include hand-edited raw keys that no code ever calls `GConfig->Set...` on. The host
process did still pick up `DefaultPlatformService=Steam` and `bEnabled=true` early enough to become
the `Steam` online subsystem, but by the time the accepted connection's `PacketHandler::Initialize`
read `[PacketHandlerComponents] Components` (at connection-accept, well after launch), the section
was gone: no `Loaded PacketHandler component: OnlineSubsystemSteam.SteamAuthComponentModuleInterface`
line, and no `Unable to Load Module`/`Unable to load HandlerComponent factory` failure line either —
the entry was silently never seen. This means the fix's correctness rests on the engine source
reading above (`PacketHandler.cpp:441-470`, confirmed against the engine's own convention at
`NetConnection.cpp:762`) and the passing automation test
(`SandboxARPG.Server.SteamIdentity.DottedFactoryNameResolvesBareNameDoesNot`, which resolves the
same `StaticLoadClass` call against the engine's always-present
`Engine.EngineHandlerComponentFactory` headlessly, without Steam), not on this live run — the live
run is blocked by this config-persistence defect, reported as found rather than worked around.

Second, in that same run: the joiner's own online subsystem defaulted to `NULL`, not `Steam`, with
no error logged. At the time this read as a second, independent module-load-ordering race — but a
third run the same evening (below) shows it was the *same* root cause as the config-persistence
defect above, not two separate ones: the joiner process was launched roughly forty seconds after the
host, by which point the host's own presence in the same shared `Saved/Config/WindowsEditor/Engine.ini`
had already been pruned back to nothing (confirmed separately: the file was found completely deleted
within about six seconds of a solo host launch, no joiner involved). The joiner therefore loaded that
file's `DefaultPlatformService=Steam` too late to matter — or never saw it at all — and fell back to
`NULL`. Both processes read the identical Saved-layer file; only its timing relative to each launch
differed.

**Third run, tracked config, root cause resolved (2026-09-23, same evening).** Since a Saved-layer
override is not durable mid-session, this run used a **temporary, uncommitted edit** to the worktree's
tracked `Config/DefaultEngine.ini` (`[OnlineSubsystem] DefaultPlatformService=Steam`,
`[OnlineSubsystemSteam] bEnabled=true`, the dotted `[PacketHandlerComponents]` line) and
`Config/DefaultGame.ini` (`bSteamAuth=True`) — both processes read this same tracked layer at launch,
before anything has a chance to prune it, and it was reverted with `git checkout -- Config/`
immediately after (`git status` clean before and after). Host and joiner alike logged
`LogOnline: STEAM: [AppId: 480] Client API initialized 1` and `OSS: Created online subsystem instance
for: Steam`; both accepted connections logged
`PacketHandlerLog: Loaded PacketHandler component: OnlineSubsystemSteam.SteamAuthComponentModuleInterface ()`
(no failure line). The host's own local login logged the usual `PostLogin` "ticket validation
pending" line, then — for the joiner's connection — the real ticket result:

```
LogOnline: STEAM: AUTH HANDLER: Sending auth result to user 76561198078624306 with flag success? 1
LogNet: Login request: ?Name=BungleBonk userId: STEAM:BungleBonk [0x1100001070E0232] platform: STEAM
LogSandboxARPG: Display: MonsterGameMode: BP_TopDownController_C_1 logged in; Steam id 76561198078624306, ticket validation pending (the engine kicks this connection if it fails)
LogNet: Join succeeded: BungleBonk
```

No kick followed. `Slime.ShowSteamIdentity` on the host afterward printed
`bSteamAuth=1 steamIsActiveSubsystem=1`, `BP_TopDownController_C_0: id=76561198078624306 validated=1`,
`BP_TopDownController_C_1: id=76561198078624306 validated=1` — a real, validated two-process Steam
join. Both processes carry the *same* Steam id because they share one logged-in Steam client on one
PC (there is no second account available for this check, per `DS-0.4` §5.4); Steam's own
`BeginAuthSession` did **not** refuse the second session as a duplicate or same-user login — the
anticipated "quote the refusal" finding did not occur, and is recorded here as such rather than
assumed.

**Contract C-7 stays open.** The registration path is confirmed correct, `IsValidated()` reflects it,
and a real ticket has now been shown validated end-to-end on this machine. What remains owed is the
real second-account proof `DS-0.4` §5.4 already names (a forged or expired ticket refused, and two
genuinely distinct Steam accounts) — closing C-7 from a single shared account is a human call, not
this check's to make. Full logs are kept outside the repository at
`C:\Users\jeffr\.claude\scratch\s-0-3\logs\` (first check) and
`C:\Users\jeffr\.claude\scratch\s-0-3\logs2\` (later sessions, including this run).

The headless automation run passed `-nosteam`, which short-circuits the online subsystem module
before any of this loads.

Packaging picks up one artefact change for everyone regardless of these switches:
`Steamworks.build.cs` stages `steam_api64.dll` into every packaged build now that the plugins are
enabled, whether or not Steam is ever turned on at runtime. The next `playtest-*` package will
contain it.

## What is not built here

No GSLT logon (`DS-0.4` §5.2 item 2, a small follow-up `S-` row once the App ID exists), no ban or
ownership lookup through the Steam Web API, no persistence keyed to this id (`S-2.4`, blocked on
`DS-2.1` and `CLAUDE.md` 1.9), no second-account validation proof (owed to a real two-developer
session, per `DS-0.4` §5.4).
