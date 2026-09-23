# Replication

*Enabled in roadmap item S-0.4. Filtering, prioritization and object-count sizing audited in S-2.3a. Design rules: GDD 9.2, the networking system; GDD 7.7, the server decides. Decision: `docs/decisions/DS-0.2.md`.*

## What it is

How the server's version of the world reaches each player's screen. The game uses Unreal's newer system, **Iris**, chosen because it is built for a hundred players in one place and because it can be switched off with one setting if it ever misbehaves.

## Where it lives

`Config/DefaultEngine.ini`, `[SystemSettings]`:

| Setting | Value | What it does |
|---|---|---|
| `net.Iris.UseIrisReplication` | 1 | Selects Iris. **Set to 0 to fall back to the classic driver** — nothing else changes |
| `net.IsPushModelEnabled`, `net.Iris.PushModelMode` | 1 | Properties announce when they change instead of being scanned every tick |
| `net.SubObjects.DefaultUseSubObjectReplicationList` | 1 | Components declare themselves for replication explicitly |
| `AbilitySystem.Fix.ReplicateTagCountContainerWithIris` | **0** | Governs how ability tags reach players under Iris. Tested both ways on 2026-09-14: 0 works, 1 silently delivers nothing. Do not change |

The Iris plugin is enabled in `SandboxARPG.uproject`, and `SandboxARPG.Build.cs` calls `SetupIrisSupport(Target)`.

## What every networked class has to do

The rules Jon's code follows so that the fallback stays a one-line switch — contract C-3 in `ROADMAP-SERVER.md`, spelled out in DS-0.2 §5. The short version: register your components for replication, mark properties dirty when you change them, and never decide who gets what — that is configured centrally, not in gameplay code.

## How we know it works

Two dev-only console commands, `Slime.IrisTagWatch` (run on a client) and `Slime.IrisTagTest` (run on the server), toggle an ability tag on a player's character and count how many times the change arrives. On 2026-09-14: Iris delivered 8 of 8 changes, in the same frame each time; the classic driver delivered 8 of 8 with four arriving a frame later. Same build, same map, one setting changed.

## Who sees what, and how big the buffers are (S-2.3a)

Nobody in this project's code decides relevancy, filtering or priority (C-3 rule 4 — no `IsNetRelevantFor`, no `bAlwaysRelevant`, no `NetCullDistanceSquared` tuning by gameplay classes). That is deliberate: it is Jeff's config, not Jon's code, so the rules can change without touching a class. Today that config is the engine's own default, verified rather than assumed (`CLAUDE.md` 7.6):

- **Filtering.** `UEngineReplicationBridge::ShouldSpatialize` (`EngineReplicationBridge.cpp:243`) puts any actor class on the default Spatial filter (`BaseEngine.ini` `DefaultSpatialFilterName=Spatial`) unless its CDO sets `bAlwaysRelevant`, `bOnlyRelevantToOwner` or `bNetUseOwnerRelevancy`. None of this project's classes sets any of the three — a headless test (`SandboxARPG.Server.Replication.NoRelevancyOverrideInGameplayCode`) sweeps every `SandboxARPG` actor class and fails if one ever does. The two actor families that carry a flag inherit it from the engine's own base class, not from this project: `AGameStateBase` (`bAlwaysRelevant`, `GameStateBase.cpp:26` — "the only global actors always-relevant") and `AController` including `AMonsterAIController` (`bOnlyRelevantToOwner`, `Controller.cpp:67` — inert for an AI controller, which owns no connection).
- **Prioritization.** No project-specific `PrioritizerConfigs` entry exists; every spatially-filtered object (pawns, monsters, projectiles, pickups, the gate) falls back to the engine's own default spatial prioritizer, `DefaultPrioritizer` (`SphereNetObjectPrioritizer`, `BaseEngine.ini`), and `PlayerState` keeps the engine's own `PlayerStatePrioritizer` override. Nothing to add at this scale.
- **Dormancy.** Nothing in this project calls `SetNetDormancy` / `FlushNetDormancy`. DS-0.3 does not currently name a case where gameplay code should (C-3 rule 4 only allows it "where DS-0.3 says so") — a candidate (a `SoulGate` that has opened, a `GemPickup`/`SoulPickup` sitting idle before it is taken) is left for a DS-0.3 addendum and a follow-up item, not invented here.
- **Object pool.** `[/Script/OnlineSubsystemUtils.IpNetDriver]` `ReplicationSystemConfigServer` / `ReplicationSystemConfigClient` in `Config/DefaultEngine.ini` (the `UPROPERTY(Config)` pair on `UNetDriver`, `NetDriver.h:857-861`; a 0 means the engine default, `NetDriver.cpp:299`) set `MaxReplicatedObjectCount` to 16384 on both sides instead of the engine's 65536 (`ReplicationSystem.h:98`). Two facts drive the number. Every replicated subobject is an object of its own under Iris (`FNetRefHandleManager::AddSubObject`, `NetRefHandleManager.cpp:746`), so a player pawn costs seven slots (pawn, ASC, attribute set, soul, progression, gem and tree components) plus its PlayerController and PlayerState, and a field monster seven. And running out is fatal, not a warning: `NetRefHandleManager.cpp:312` aborts the process with "Hit the maximum limit of active replicated objects". Counted on the slice, GDD 7.3's 100 players and MobDensity at its validated ceiling of 10 on the spawn table's rows come to about 2500 objects; 16384 is six times that and covers an open map with tens of spawners per zone, and both the per-object lists and the preallocated chunked buffers are clamped to it (`NetRefHandleManager.cpp:108`, `114`), which is where the saving over 65536 is. The client gets the same value because in a full town the Spatial filter can send it most of the server's set. The loaded value is logged at start (`NetRefHandleManager: Configured with MaxActiveObjectCount=16384`); a headless test holds it below the default and above twice the counted worst case. The number is a placeholder until S-2.6 measures the real count and DS-2.3 sets it (`PROVISIONAL(DS-2.3)` on the config line, register row).
- **RPC validation.** All twelve `Server, Reliable, WithValidation` RPCs in the project (`SoulComponent` ×7, `GemEquipmentComponent` ×2, `SoulTreeComponent` ×2, `MonsterCharacter` ×1) have a real `_Validate` — an audit finding, not new code this item. What `WithValidation` buys depends on the driver, and this was read, not assumed: the UHT thunk returns before `_Implementation` when `_Validate` is false on either driver (`SoulComponent.gen.cpp`, `RPC_ValidateFailed`), so a malformed call never runs. The disconnect DS-0.3 §1 promises ("`_Validate` returning false disconnects the sender") happens only on the legacy driver, where `FObjectReplicator::ReceivedRPC` reads `RPC_GetLastFailedReason` and fails the bunch (`DataReplication.cpp:1465`) and `UActorChannel::ProcessBunch` closes the connection (`DataChannel.cpp:3472`). Iris's RPC path (`NetRPC.cpp:730`) calls `ProcessEvent` and never reads the reason: under the project's driver the call is dropped silently, nothing is logged and the S-1.5 kick does not see it. Filed as S-1.6. Per-connection rate limiting (DS-0.3 §1, "values live in the server-rule table") is `E-2.11`, still `todo`, and is not part of this item.
- **GAS tag-change delegate.** Already tested on the pinned 5.8 build in DS-0.2 §4a (2026-09-14): `AbilitySystem.Fix.ReplicateTagCountContainerWithIris=0` passes, `1` silently drops every change. Unchanged here.

## Owed to S-2.3b

Per-property conditioning and delta serialization: `OwnedSouls` on `USoulComponent` (PR #27) and the twelve `UMonsterAttributeSet` attributes (eight in PR #29, the four typed resistances since) all replicate `COND_None` today and should condition to the owner where no other player needs the value; `OwnedSouls` should move to a FastArray. That item also settles whether GAS's own attribute accessors route through `MARK_PROPERTY_DIRTY` or need the C-3 convention-3 exception recorded. It touches Jon's classes, `USoulComponent` and `UMonsterAttributeSet`; Jeff took it on 2026-09-23 under the standing authorisation for a blocking item, so it is not waiting on Jon.

<!-- PROVISIONAL(DS-2.3): MaxReplicatedObjectCount 16384 on server and client is a placeholder from GDD 7.3's 100-player ceiling at MobDensity 10 on the slice's zones; S-2.6 measures the real count and DS-2.3 sets the value -->

Also settled while reading for this item: the C-3 rule that every replicated actor uses the registered subobject list holds project-wide through `net.SubObjects.DefaultUseSubObjectReplicationList=1` (`Config/DefaultEngine.ini`), which Iris itself requires (`EngineReplicationBridge.cpp:234`, an `ensure` at bridge start).

## What is settled and what is waiting

- **Settled:** Iris is the driver; the tag cvar stays at 0; the fallback is real; filtering, prioritization and dormancy for this project's classes are audited and, at today's scale, need no override beyond the object-count buffer.
- **Owed:** per-property conditioning and the `OwnedSouls` FastArray (S-2.3b, above); a dormancy policy for idle world actors, if DS-0.3 ever asks for one.
- **Watch for:** a warning about "2 active ReplicationSystems" appears in editor two-window testing and in two `-game` processes started from the same machine. It is an artefact of running more than one `ReplicationSystem` in one process (the process's own single-player world before `Slime.Host`/`Slime.Join`, then the networked one) and should not appear on a real dedicated server. If it does, that is a finding.
