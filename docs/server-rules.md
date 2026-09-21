# Server rules

*Built in roadmap item S-1.1 (pull request #75). Design rules: GDD 8.1, the list of rules a host can set; GDD 7.4, servers are player-hosted; GDD 7.7, the server decides.*

## What they are

Whoever hosts a server sets the rules for it. Twelve of them exist today, matching the GDD's list:

| Rule | What it does | Kind |
|---|---|---|
| Hardcore | Death is permanent | on / off |
| Drop all loot on death | Everything you carry drops when you die | on / off |
| Equipment breaks | Gear can wear out | on / off, inert until the durability design lands |
| Crafting forgiveness | How forgiving crafting is; 1.0 is baseline | scale |
| PvP | Whether players can hurt each other | on / off — see [below](#getting-a-rule-to-a-client-the-pvp-flag-on-the-gamestate-s-32a) for how a client learns it |
| Boss difficulty | Scales bosses; 1.0 is baseline | scale |
| Boss moveset | Which set of moves bosses use | choice |
| Field monster difficulty | Scales everything else; 1.0 is baseline | scale |
| Field monster moveset | Which set of moves field monsters use | choice |
| Mob density | How many monsters spawn; 1.0 is baseline | scale |
| Drop rates | How likely drops are — the chance, never the count | scale |
| Guild size | Solo, pairs, or three and up | choice |

## Where they live

In a text file the host edits, not in the game's code. The project ships baseline values in `Config/DefaultGame.ini`; a host overrides any of them in the server's own config layer. Scales run from 0.1 to 10 (density and drops from 0), so a host cannot accidentally make a monster hit a thousand times harder.

## What happens if the host types something wrong

The server logs a warning naming the rule, the bad value and the range, and keeps the built-in default for that one rule. It never rewrites the host's file and never silently overrides the rest of it — two things other survival games get wrong, and the reason this exists as its own system.

## Rate limits on what a client may ask for

Every request a player's machine can send the server — activate a move, equip a soul or gem, allocate a skill point, choose a respawn — has a minimum gap between two sends from the same player. Six settings in the same file cover them (one per kind of request plus a default), and a request that is not in the table gets the default and is logged once so someone adds it. Sending faster than the gap is dropped and logged, never queued — a stuck key or a bad actor cannot flood the server.

## How the game reads them

Through one door: `USandboxServerRules::Get()`. The spawner asks it for mob density, the loot roll asks it for drop rates, damage asks it whether PvP is on. Nothing in gameplay code reads a rule from anywhere else, so there is exactly one place a rule can be wrong. Server-side only — a client that needs to display a rule gets it through replicated game state, never from its own copy of the file.

`Slime.ShowServerRules` prints every rule as the server currently holds it.

## Getting a rule to a client: the PvP flag on the GameState (S-3.2a)

The rules above live in `USandboxServerRules`, which is server-side only — a client's own copy of `Config/DefaultGame.ini` is not authority. The PvP rule is the first one a client needs to see (for name-plate colour and, later, hostility), so it gets its own carrier: `AMonsterGameState`, a small `AGameStateBase` subclass that holds nothing but that mirror.

`AMonsterGameState` replicates one property, `bPvP`, push-model (`ReplicatedUsing = OnRep_PvP`, `FDoRepLifetimeParams.bIsPushBased = true`, `DOREPLIFETIME_WITH_PARAMS_FAST`), the same convention other push-model flags in the project use (`bInSafeZone`, per the STATUS notes). `SetPvP(bool bEnabled)` is authority-only — it returns at once if `!HasAuthority()` — and marks the property dirty with `MARK_PROPERTY_DIRTY_FROM_NAME` after setting it. Because the engine gives the authority machine no `OnRep` call of its own, `SetPvP` calls `OnRep_PvP()` directly so the host's log line matches every other machine's. `OnRep_PvP` logs:

```
MonsterGameState: PvP %d
```

(`1` or `0`), at `LogSandboxARPG`, `Log`. `IsPvP() const` is the public accessor everything else should read.

`AMonsterGameMode` installs the class in its constructor (`GameStateClass = AMonsterGameState::StaticClass();`, beside its existing `DefaultPawnClass`/`HUDClass` lines) and fills it once, in a new `InitGameState()` override: after `Super::InitGameState()`, it fetches the game state as `AMonsterGameState` and calls `SetPvP(USandboxServerRules::Get().bPvP)`. If the game state is not an `AMonsterGameState` — which should not happen outside a misconfigured project — it logs instead of setting anything:

```
MonsterGameMode: GameState is not AMonsterGameState; PvP flag not set
```

The ini row this reads is unchanged and unremarkable: `Config/DefaultGame.ini`, `[/Script/SandboxARPG.SandboxServerRules]`, `bPvP=False`.

**What reads it today: nothing.** The flag exists and replicates, but no gameplay code calls `IsPvP()` yet. `AMonsterCharacter::IsHostileTo` still hardcodes another player as never hostile, regardless of the flag, and the name-plate colour path (`NamePlateWidget`) inherits that gap — both are Jon's side of this item. A joining client actually receiving the replicated value has not been tested; only a Standalone host toggling its own config and reading its own log has (`bPvP=True` logs `PvP 1`, `bPvP=False` logs `PvP 0`). S-3.2a stays open until that two-player check runs and the pawn side lands.

## Repeat-failure kick (S-1.5)

DS-0.5 §3's anti-cheat choice — server validation only, no kernel-level client anti-cheat in pre-alpha — adds one policy the server enforces itself: a connection that keeps failing the same kind of server check gets disconnected. `UFailureKickSubsystem`, a `UWorldSubsystem`, keeps a rolling per-connection list of server world-time stamps and prunes anything older than the window on every new failure.

**What counts.** Only the DS-0.3 §3 rows the memo marks *logged*: row 3 (ability ownership, `SoulComponent.cpp`, a move the equipped soul does not have — this branch also fires when no soul is equipped at all), row 8 (item ownership, `GemEquipmentComponent.cpp`, equipping a gem that is not in the bag) and row 13 (menu-choice legality, `SoulTreeComponent.cpp`, an illegal skill-tree allocation) — each calls `RecordFailure` right after its existing `UE_LOG` line. Row 2 (teleport discontinuity) and row 14 (the RPC rate limit) join once their own logging exists; neither is built yet.

**What never counts.** DS-0.3 §3's *corrected* rows (1, 4-7, 9-12): a lagged client can hit those honestly, and the server just corrects it silently, never logs it, so `RecordFailure` is never called for them. Tested 2026-09-21: a lagged client re-sending a stale `Slime.EquipGem` counts toward the kick (row 8, DS-0.5 logged rows count regardless of why the client sent it), while repeated cooldown refusals (row 4, corrected) never do.

**The two rules.** `FailureKickThreshold` (default 5) and `FailureKickWindowSeconds` (default 30), both `PROVISIONAL(DS-1.1)` like every other 8.1 default, in the same `USandboxServerRules` file and the same `Config/DefaultGame.ini` section as the rest. Crossing the threshold inside the window clears that connection's record and kicks it.

**The log lines, verbatim:**

```
FailureKick: %s kicked after %d failure(s) in %.0f s (last: %s) net id %s (S-0.3 not landed: net id, not a validated Steam ID)
```

```
FailureKick: %s is the listen host; %d failure(s) in %.0f s (last: %s) not kicked
```

**The listen-host exception.** A local controller (the host's own player on a listen server) is never kicked — there is nowhere for it to go — and the subsystem logs the warning instead.

**The net-id caveat.** The kick log names `APlayerState::GetUniqueId().ToString()`, which today is the machine's net id, not a validated Steam ID (S-0.3 has not landed). The log line says so plainly rather than implying more identity than the project actually has.

**Nothing persisted.** No ban list, no strike record, no save write — the count lives only in the subsystem's `TMap` and is gone with the world (`CLAUDE.md` 1.9).

**Dev command.** `Slime.ShowFailures` prints the threshold and window, then one line per tracked connection: `FailureKick: <name> <count> failure(s) in the window`.

## What is settled and what is waiting

- **Settled:** the list of rules and the way they are read (S-1.1, contract C-2).
- **Provisional:** every default value. GDD 8.1 says defaults are open until tuned; the ones shipped are baseline placeholders, registered under DS-1.1.
- **Open:** whether guild size caps a party, a guild, or both — nothing reads it until that is decided. Which combinations of rules are locked together (GDD 8.2). What "equipment breaks" means.
- **Owed:** the warning currently fires the first time something reads a rule, not at startup. A host with a typo learns about it late. Roadmap item S-1.3 moves the check to server start.
