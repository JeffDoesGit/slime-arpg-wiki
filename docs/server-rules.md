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

In a text file the host edits, not in the game's code. The project ships baseline values in `Config/DefaultGame.ini`; a host overrides any of them in the server's own config layer. Scales run from 0.1 to 10 (density and drops from 0), so a host cannot accidentally make a monster hit a thousand times harder. For the copy-pasteable ini block, the full key table with ranges and readers, and the override-file path for each way of hosting, see [`../server-config.md`](../server-config.md).

## What happens if the host types something wrong

The server logs a warning naming the rule, the bad value and the range, and keeps the built-in default for that one rule. It never rewrites the host's file and never silently overrides the rest of it — two things other survival games get wrong, and the reason this exists as its own system.

## Rate limits on what a client may ask for

Every request a player's machine can send the server — activate a move, equip a soul or gem, allocate a skill point, choose a respawn — has a minimum gap between two sends from the same player. Six settings in the same file cover them (one per kind of request plus a default), and a request that is not in the table gets the default and is logged once so someone adds it. Sending faster than the gap is dropped and logged, never queued — a stuck key or a bad actor cannot flood the server. Since E-2.11 the game reads these: every client-to-server RPC (the move, the six soul equips and unequips, the two gem ones, allocate and respec, the respawn choice) checks `AMonsterCharacter::AllowServerRpc` first, on the host, which remembers when it last accepted that request from that pawn and drops one that comes sooner than the table's gap with a line like `RateLimit: MonsterCharacter_1 ServerRequestActivateMove dropped (0.031 s since the last, minimum 0.10 s)`. A dropped request is not a failure for the repeat-failure kick; a fast client is throttled, not kicked. In ordinary play nothing is dropped: the hold-to-repeat and the input buffer already send slower than the move gap. A tester who wants to see it can mash `Slime.UseMove 0` through the debug menu's command box on the host.

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

**What reads it today (S-3.2b, 2026-09-23): the pawn side.** `AMonsterCharacter::IsHostileTo` calls `GetWorld()->GetGameState<AMonsterGameState>()->IsPvP()`: another player-controlled pawn is hostile only while that flag is on and both pawns are player-controlled (`PROVISIONAL(D-3.6)` rule 6, "PvP ... is a multiplier row applied at step 4, never a second formula"; the D-3.6 §6 placeholder — `PvP damage multiplier 1.0` — is now `USandboxServerRules::PvPDamageMultiplier`, applied by `ApplyHit` at the same step as the ratio-mitigation formula, when both attacker and target are player-controlled). Safe zones are unchanged: `ApplyHit`'s existing gate (checked first, above the mitigation steps) already refuses a hit on or from a safe-zoned pawn, PvP included, and this is not a second check. The name-plate colour (`NamePlateWidget`) re-picks on a viewer change, a PvP-flag change, *or* the owner's own `IsPlayerControlled()` changing (so a `PlayerState` that replicates in after the plate's first tick still recolours it), so two players' plates flip live when the host toggles the rule; the cursor-aim log now reads `aim: player <name>` instead of `aim: monster <name>` when the aim target is a player-controlled pawn (the internal aim-source sentinel is unchanged, cosmetic only). Two-player proof, editor-hosted listen host and joiner, `bPvP=True`, `PvPDamageMultiplier=2.0`, both wearing the Creepy: `Hit: MonsterCharacter_0 hit MonsterCharacter_1 for 20.00 Physical (100.0 -> 80.0)` outside a safe zone; `Hit: MonsterCharacter_0 on MonsterCharacter_1 not applied (safe zone)` inside the town's; with `bPvP=False`, the same dash crosses the other pawn with no `Hit:` line at all (`Dash: ... ended, crossed 0`). `Data/DT_CombatRules.csv` still carries a `PvPMultiplier` row; it is not read anywhere — the live number is `USandboxServerRules::PvPDamageMultiplier` (DS-1.1 §3c). **What this does not do:** no party immunity (no party system exists — every AoE move a player has, not only a single-target one, now also lands on any hostile player while PvP is on), no opt-in, no loot or XP change (a player's death path never reads who killed it), and the pre-existing safe-zone-flag leak on a level unload (`bInSafeZone` never clears on unload, ROADMAP "Known gaps") would leave a leaked pawn PvP-immune too — not fixed here. **Known gaps (S-3.2b):** the dead `PvPMultiplier` row in `DT_CombatRules` waits to be retired under lock (content, a human, 7.8).

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

## Drop all loot on death (S-3.1, drop-all half; Hardcore still blocked)

**What reads it today (S-3.1, 2026-09-23): a player's own death path.** `AMonsterCharacter::HandleHealthDepleted` calls the pure gate `AMonsterCharacter::EvaluateDropAllOnDeath(bDropAllLootOnDeathEnabled, bDyingPawnIsPlayerControlled)` before the respawn timer starts; when it opens, `UGemEquipmentComponent::TakeAllGems()` empties the bag and every equipped slot (removing each equipped gem's modifiers on the way off, same as an ordinary unequip) and returns what it took, and `FGemDrops::SpawnPickupForGem` puts each one on the ground as an ordinary `AGemPickup`, scattered 150 uu around the corpse in a ring so the dead pawn's own capsule — still standing there until respawn — cannot overlap and re-grant a pickup to itself the instant it spawns (the first runtime check of this row found exactly that: a `GemDrop: ... dropped ... on death` line immediately followed by `GemPickup ...: taken by` the same pawn, from the pickup landing under its own capsule). PROVISIONAL(DS-1.1): what "drop all loot" means is DS-1.1 §5.1's recommendation, quoted in full below.

**What never drops.** Souls, soul fragments, tree points, level and XP are untouched — the drop-all step only ever calls into `UGemEquipmentComponent`, which has no reach into any of those (DS-1.1 §5.1: "Souls are account unlocks ... never lost, consumed or displaced" — GDD 2.8 `DECIDED`). A field monster's own death keeps its separate drop roll (E-2.32) regardless of this rule; the gate is player-deaths only.

**Quoted recommendation (DS-1.1 §5.1, "What 'drop all loot on death' includes"):** *"the bag's gems and every equipped gem drop at the corpse as pickups; souls, soul fragments, tree points, level and XP never drop... Whether an equipped gem drops or only the bag's is the one sub-question; the recommendation is both, because 'all' is the word in the row and a rule that spares equipped gems is the softer rule a host would want named differently."*

**Hardcore, the other half of this row, stays out.** DS-1.1 §3a: "It also cannot be honoured before persistence exists (S-2.4): 'permanent' is a save-record fact (1.9), so a build with no saves has nothing to make permanent." `CLAUDE.md` 1.9 forbids provisional code from writing anything a later session reads, and Hardcore's whole meaning is a save-record fact, so no code for it ships here. S-3.1 stays `provisional` rather than `done` until S-2.4 lands and the Hardcore half is built.

**Pure functions, tested headless.** `AMonsterCharacter::EvaluateDropAllOnDeath` (the gate) and `UGemEquipmentComponent::SelectGemsToDrop` (what the selection is: every valid equipped-slot gem, then every bag gem, in that order — pulled out of `TakeAllGems()` so it has no world dependency) are the same shape as `EvaluatePvPHostility` (S-3.2b) and `ClampListenMaxPlayers` (S-2.2). Six automation tests, `SandboxARPG.Server.DropAll.*`: the gate off, on with a player, on with a non-player (unaffected); the rule reads without asserting; the selection is empty with nothing carried; the selection includes both an equipped-slot gem and a bag gem with no duplicate and no empty-slot entry.

**Runtime check, Standalone listen host, `Lvl_Slice`.** With `bDropAllLootOnDeath=True` in the user config layer: granting a Red and a Green gem, then `Slime.SetAttribute Health 0`, logged `GemDrop: MonsterCharacter_0 dropped Red gem ... on death (drop-all)`, the same for Green, `MonsterCharacter_0: drop-all on death: 2 gem(s) left at the corpse`, and `Slime.ShowGems` five seconds later read `bag 0, slots 9` — down from `bag 2, slots 9` before death. With the key removed (default `False`): the same sequence left `bag 2, slots 9` unchanged after death, no `GemDrop` or `drop-all` line.
## A failed `_Validate` is logged and counted the same way under Iris (S-1.6)

Every one of the project's twelve Server RPCs is `WithValidation`: the UHT thunk calls its `_Validate` first and, on false, never runs `_Implementation`. What happens next used to depend on the driver. The legacy `IpNetDriver` reads the `RPC_ValidateFailed` reason in `FObjectReplicator::ReceivedRPC` (`Engine/Private/DataReplication.cpp:1465`) and closes the connection outright. Iris's own RPC receive path (`Engine/Source/Runtime/Net/Iris/Private/Iris/ReplicationSystem/NetBlob/NetRPC.cpp:730`) never reads that reason — a failed `_Validate` under Iris was previously dropped with no log line and no disconnect, so DS-0.3 §1's promised disconnect held only on the fallback driver, and the S-1.5 kick above (which only counts *logged* failures) never saw an Iris client that kept sending bad RPCs.

**The fix.** `UFailureKickSubsystem::ReportValidateFailure(AActor* Owner, FName RpcName)` is a static helper every `_Validate` false path calls: it resolves the sending `PlayerController` from the RPC's owner (the pawn itself for `AMonsterCharacter::ServerRequestRespawn`, or the owning pawn for a component RPC), logs `RPCValidate: <Rpc> failed validation, sent by <player> (owner <actor>)`, and calls `RecordFailure` exactly as the three DS-0.3 §3 rows above already do. Nine of the twelve `_Validate` bodies actually have a false path and call it (`USoulComponent::ServerRequestEquipSoul` / `EquipSecondarySoul` / `EquipWornSoul` / `ActivateMove`, `UGemEquipmentComponent::ServerRequestEquipGem` / `UnequipGem`, `USoulTreeComponent::ServerRequestAllocate` / `ServerRequestRespec`, `AMonsterCharacter::ServerRequestRespawn`); the other three (`UnequipSoul`, `UnequipSecondarySoul`, `UnequipWornSoul` — no arguments) always return true and were left untouched. No `_Validate` body's pass/fail logic changed.

**Dev command.** `Slime.SendBadRpc`, run on a client (every other `Slime.*` soul command here runs on the authority and calls the gameplay function directly), sends `ServerRequestActivateMove(nullptr, NaN)` — a null ability and a NaN aim, the one input no legitimate caller can construct — straight over the wire.

**Proven both ways, two-process, `net.Iris.UseIrisReplication` 1 and 0:**
- **Iris:** six `Slime.SendBadRpc` from the joiner produced five `RPCValidate: ServerRequestActivateMove failed validation, sent by BP_TopDownController_C_1 (owner MonsterCharacter_1)` on the host, then `FailureKick: BP_TopDownController_C_1 kicked after 5 failure(s) in 30 s`. The joiner's own sixth attempt logged `Slime.SendBadRpc: MonsterCharacter_0 is the authority` — it had already reverted to a fresh local pawn after the kick. In the kick frame the host also logs three or four `LogIrisRpc: Error: Rejected RPC (0:16) ... due to missing object or function` lines (one of them the joiner's `ServerMovePacked`): RPCs already received for the destroyed pawn, dropped by the reader, harmless. The host spawns `MonsterCharacter_2` at the kick, the same third pawn `ROADMAP.md`'s known gaps record for a joiner killed mid-session (the legacy disconnect does it too); it is not part of this change.
- **Legacy:** the very first bad RPC produced one `RPCValidate:` line, immediately followed by the engine's own `LogNet: - Result=ObjectReplicatorReceivedBunchFail` and `UChannel::Close: Sending CloseBunch` — DS-0.3 §1's disconnect is unchanged, fires before a second failure is ever recorded, and no crash or double count followed.

**Why the kick may run inside `_Validate`.** `KickPlayer` destroys the pawn and the controller before it returns, and it runs here from the driver's receive path. That is the same call depth as the three S-1.5 kicks above, which run from code reached through `_Implementation` (the UHT thunk calls both from the same place). The legacy driver expects it: `UActorChannel::ProcessBunch` names an actor destroyed by a client-to-server RPC "a legitimate occurrence" and only marks the channel broken (`Engine/Private/DataChannel.cpp:3479`). Iris tolerates it: `FNetRPC::CallFunction` touches nothing but its parameters after `ProcessEvent`, the reader then pops its own attachment queue, and `FNetRefHandleManager::DestroyNetObjectByIndex` defers the net object's destruction ("We always defer the actual destroy"). So the kick is not deferred to the next tick; the engine citations sit on the `RecordFailure` comment in `FailureKickSubsystem.h`.

**Headless tests** (`Source/SandboxARPG/Server/Tests/ValidateFailureTest.cpp`, `SandboxARPG.Server.ValidateFailure.*`): a bare `UWorld::CreateWorld` Game world with a spawned pawn and player controller, no GameMode, so the threshold logs the kick decision and stops at `could not be kicked; no GameSession`. They show one `RPCValidate:` line and one counted failure per call, the count dropped and the kick line written at the threshold, and an unpossessed pawn logged but never counted. The live kick is the two-process run above.

## What is settled and what is waiting

- **Settled:** the list of rules and the way they are read (S-1.1, contract C-2).
- **Provisional:** every default value. GDD 8.1 says defaults are open until tuned; the ones shipped are baseline placeholders, registered under DS-1.1.
- **Open:** whether guild size caps a party, a guild, or both — nothing reads it until that is decided. Which combinations of rules are locked together (GDD 8.2). What "equipment breaks" means.
- **Settled, corrected 2026-09-21:** validation is still lazy — it runs the first time anything calls `USandboxServerRules::Get()` — but S-1.3 (PR #202, `a9d6bbb`) made `AMonsterGameMode::InitGame` call `Get()` itself, before the first player can connect. On every launch path that runs through `AMonsterGameMode::InitGame`, a typo is now caught and logged at server start, not whenever some unrelated system first happened to read a rule. What has actually shown this so far: PIE and the editor-hosted `-game` listen host — the only paths tested. Whether that same call fires on a packaged, non-editor dedicated-server launch path is untested — there is no dedicated server target in this project yet (`ROADMAP-SERVER.md` §3 STATUS).
