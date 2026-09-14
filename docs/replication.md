# Replication

*Enabled in roadmap item S-0.4 (pull request pending). Design rules: GDD 9.2, the networking system; GDD 7.7, the server decides. Decision: `docs/decisions/DS-0.2.md`.*

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

## What is settled and what is waiting

- **Settled:** Iris is the driver; the tag cvar stays at 0; the fallback is real.
- **Owed:** who sees what — right now every player receives every other player's full state, which is fine at five players and wasteful at a hundred. That tuning is roadmap item S-2.3.
- **Watch for:** a warning about "2 active ReplicationSystems" appears in editor two-window testing. It is an artefact of running server and client in one process and should not appear on a real dedicated server. If it does, that is a finding.
