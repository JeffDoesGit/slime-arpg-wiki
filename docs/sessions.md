# Sessions

*Built in roadmap item S-2.2 (contract C-5), provisional on [`docs/decisions/DS-2.2.md`](../decisions/DS-2.2.md) §5.2 until Jeff records the hosting-tier decision in the GDD. Design rules: GDD 7.1 (single player), 7.2 (local co-op), 7.3 (100-player cap), 7.4 (player-hosted, configurable).*

## What it is

Every session the game has ever run — single player, a friend joining, the playtest with three
people on Tailscale — is a **listen host**: one process that renders and simulates at once, the
others connecting to it. There is no dedicated server binary yet ([S-2.1](../../ROADMAP-SERVER.md), [`DS-0.10`](../decisions/DS-0.10.md)), so hosting, joining and playing alone are three
uses of the same code path rather than three different modes.

`USessionSubsystem`, a `UGameInstanceSubsystem` in `Source/SandboxARPG/Server/`, wraps the
engine's own level-travel calls so a menu — Jon's `E-6.5`, not built here — can call the same
three functions the `Slime.*` console commands below already exercise, and so the same functions
keep working unchanged once a dedicated binary exists (`DS-2.2` §5.4: the binary is launched with
its map on the command line, never through `HostSession`; `JoinSession` is identical for a client
whether the address is a friend's listen host or a rented box).

## The three calls

| Call | What it does |
|---|---|
| `HostSession(FName Map, int32 MaxPlayers, bool bListen)` | `bListen=true` is the in-game host: opens `<Map>?listen?MaxPlayers=<n>`, with `n` clamped to the `ListenHostMaxPlayers` server rule. `bListen=false` is the single-player entry: the same travel with no `?listen`, so no socket ever opens — still a server world, just with one player and nobody able to join it. |
| `JoinSession(const FString& Address)` | Connects the local player to a host at `Address` (an IP, or host:port). A blank address is refused and logged; everything else about the string is the engine's own URL parsing to sort out. |
| `LeaveSession()` | Returns to the project's `GameDefaultMap` alone, with no `?listen`: a host stops listening (any joiners get the engine's own disconnect), a client disconnects from whatever it was joined to. |

Two reads and one delegate round it out: `IsHosting()` (true from a `HostSession(..., bListen=true)`
call until this game instance joins or leaves; the single-player travel does not set it),
`IsListening()` (true right now, read off the world's net mode), and `OnSessionStateChanged`
(`Idle` / `Hosting` / `Joining` / `Failed`), which a menu binds to show progress or a failure reason.

On a dedicated server, `HostSession` and `LeaveSession` refuse and log: that binary takes its map
from the command line (`DS-2.2` §5.4) and either call would travel it off its listen and drop
everyone on it. `JoinSession` refuses there too, because there is no local player controller. The
subsystem still exists on that process (a `UGameInstanceSubsystem` is created on every game
instance); it just has nothing to do.

## What it deliberately does not do

No Steam, no LAN discovery beacon, no server browser, no lobby. Discovery today is an address a
host reads off their own machine and tells the other players — the same thing every playtest has
done (`docs/playtest.md`). When Steamworks lands (`DS-0.4`, `S-0.3`), `JoinSession` gains an
overload that takes a search result instead of a typed address and `HostSession` registers the
session with Steam; none of the menu's three buttons change.

## Engine calls underneath (verified against UE 5.8 source on disk)

- `HostSession` → `UGameplayStatics::OpenLevel` (`Kismet/GameplayStatics.h:340`), the same call
  `UEngine::HandleOpenCommand` makes for the console's `open <map>?listen` (`UnrealEngine.cpp:15378`).
  `?MaxPlayers=` is read by `AGameSession::InitOptions` (`GameSession.cpp:156-159`); a connection
  past that count is refused with `PreLogin failure: Server full.` on the host and a
  `PendingConnectionFailure` on the joiner (`GameSession.cpp` `AtCapacity`, engine behaviour, not
  project code).
- `JoinSession` → `APlayerController::ClientTravel(URL, TRAVEL_Absolute)` (`GameFramework/PlayerController.h:1413`), the console's `open <ip>` path.
- `LeaveSession` reads `GameDefaultMap` out of `Config/DefaultEngine.ini`
  (`[/Script/EngineSettings.GameMapsSettings]`) through `GConfig` directly, rather than adding the
  `EngineSettings` module dependency for one string, and then travels there the same way `HostSession`
  does with `bListen=false`.
- Failures reach the subsystem through the engine's own delegates, `UEngine::OnNetworkFailure()`
  and `UEngine::OnTravelFailure()` (`Engine/Engine.h:2390`, `:2381`), bound in `Initialize` and
  unbound in `Deinitialize`. Both events are engine-wide, so a play-in-editor session with a host
  window and a client window has two subsystems hearing every failure; each forwards only the
  failures whose world context is owned by its own game instance. A refused join arrives with no
  world and the pending net driver (`UnrealEngine.cpp:16017`), so that case is matched through
  `UEngine::GetWorldContextFromPendingNetGameNetDriver`; a failure with no context at all is
  forwarded rather than dropped.

## The listen-host cap

`ListenHostMaxPlayers` (`USandboxServerRules`, default 8, range 1–16) is `HostSession`'s ceiling
on a caller's requested `MaxPlayers` — `USessionSubsystem::ClampListenMaxPlayers`, a pure static
function so it is testable with no world. It counts the host's own local player along with every
joiner: a host who sets it to 1 cannot be joined by anyone, because the host already occupies the
one seat the engine's own `AGameSession::MaxPlayers` check allows (`AGameSession::AtCapacity`
compares `AGameModeBase::GetNumPlayers()`, which counts every player controller with a player state,
the host's included, `GameSession.cpp:322-340`). PROVISIONAL(DS-2.2): 8 is the memo's placeholder
(`docs/decisions/DS-2.2.md` §5.1, "a number, not a measurement") until `DS-2.3`'s scale-spike numbers
or `DS-1.1`'s 8.1 defaults replace it. See [`../server-config.md`](../server-config.md) for the ini key.

The rule reaches the engine only through the `?MaxPlayers=` option `HostSession` builds. A host
started the old way, with the console's `open Lvl_Slice?listen` (`docs/playtest.md` sections 0
and 2), passes no such option and gets the engine's own `AGameSession::MaxPlayers`, which is 16
(`Engine/Config/BaseGame.ini`; the project sets no `[/Script/Engine.GameSession]` override). Making
the rule bind that path too would mean a project `AGameSession` or a game-mode hook, which
`DS-2.2` §5.2 does not ask for and this row does not add.

## Dev commands

Compiled out of Shipping, same pattern as `Slime.ShowServerRules` and the other dev tooling:

- `Slime.Host <map> [MaxPlayers]` — `HostSession(<map>, MaxPlayers or 8, bListen=true)`.
- `Slime.Join <address>` — `JoinSession(<address>)`.
- `Slime.Leave` — `LeaveSession()`.

The memo's §5.2 spelled these `Slime.Session.Host` / `.Join` / `.Leave`; they are shortened to
match every other `Slime.*` command on the page. The three API names the memo fixes for C-5 are
unchanged.

## Admin: kicking a player (S-5.1)

A host removes a connected player by name from their own console: `Slime.Kick <PlayerName>
[Reason...]`, compiled out of Shipping like every other `Slime.*` command on this page. It looks
up the player by `APlayerState::GetPlayerName()` (case-insensitive), refuses and logs rather than
acting when the world has no authority, the name does not resolve to a connected controller, or
the name resolves to the host's own local controller (there is nowhere for a listen host's own
controller to go). A resolved kick runs through `AGameSession::KickPlayer` (the engine's own
disconnect path, `GameFramework/GameSession.h:155`) and logs
`Kick: <name> kicked by host (<reason>)` before returning. The implementation is
`AdminKick::KickByName` in `Source/SandboxARPG/Server/AdminKick.h`.

Whitelist, ban list and RCON — the rest of S-5.1's "admin minimum" — are not built. A whitelist or
ban needs a durable identity to key entries on, which is contract C-7 (validated Steam ID),
still `open`, and somewhere to keep the list across a restart, which is S-2.4 (persistence),
still `todo`. `CLAUDE.md` 1.9 rules out writing anything provisional that a later session reads,
so neither is built provisionally against a placeholder identity or an in-memory list that a
restart would silently lose. A kick has neither problem: it is a one-shot action against a
connection that is live right now, nothing about it needs to survive the process.

**Log location.** Every `Kick:` line, like every other `Slime.*` and `LogSandboxARPG` line, goes
to the running process's own log file, not to a separate admin log:

- **An editor-hosted session** (PIE, or `UnrealEditor.exe <uproject> <map>?listen -game`):
  `<Project>/Saved/Logs/<ProjectName>.log` (this repository:
  `Saved/Logs/SandboxARPG.log`), or the file named by an explicit `-abslog=<path>` / `-log=<name>`
  argument on the command line.
- **A packaged Windows client hosting as a listen server**: `<Build>/SandboxARPG/Saved/Logs/`.
- **A dedicated server**, once one exists (`S-2.1`, blocked on `DS-0.10`):
  `<Build>/SandboxARPG/Saved/Logs/` under that binary's own working directory.

There is no rotation or separate admin-log stream today; a host tailing for `Kick:` reads the
same file every other server log line lands in.

## What is not built here

No menu UI (Jon's `E-6.5`), no Steam identity or discovery (`S-0.3`, `DS-0.4`), no replication
filtering or scale numbers (`S-2.3`, `S-2.6`), no whitelist, ban list or RCON (`S-5.1`, above) —
those are separate rows this one does not touch.
