# Field monsters

*First monster built in E-2.5 (branch `feat/field-monster-creepy`). Provisional on two design calls: the AI system (D-1.3) and whether monsters play by the player's rules (D-2.4).*

## What you see

A sword-carrying skeleton (the Corpse, from the City of Brass pack) standing in the field, chosen because you play a Creepy and a monster that looks like you is hard to read. Walk toward it and, once you are close enough, it turns and comes for you along the ground, avoiding walls. When it reaches you it claws at you twice (Grave Clutch, its kit's first move since E-2.48; it carries no sword), waits for the swing to finish, and swings again. Walk far enough away and it gives up and stands still. Its hits hurt and yours hurt it (see [Combat](combat.md)). When it dies it plays its death clip, or, for a body whose pack ships no death clip like the Creepy, goes limp and falls as a ragdoll (since E-2.78); either way it lies there for a few seconds, sinks into the ground over the last one, and is gone, on every machine at the same moment. Before this a dead monster froze in its last pose and lay around until the server removed it.

A monster never picks a dead player as its target, and drops one that dies (since E-2.75); before this a Corpse spawned beside a fallen player stood over the body attacking nothing while a live player walked past. When a monster has a target it cannot reach and stands still for three seconds, it logs once what the navigation mesh says about itself and its target, so a stalled chase names its cause instead of being a mystery. The first such line, on 2026-09-17, said neither end was on any navmesh at all, which is E-2.83.

Until fix/corpse-alert-stall (2026-09-23), a Corpse could settle just outside its own swing's real reach and stay there: the monster definition's `AttackRange` (how close counts as "arrived," read by both the chase and the attack states) was wider than the attack move's own `HitRange` (how close the swing actually needs to land, checked only when the swing resolves), so a Corpse would walk up, decide it had arrived, swing on a loop forever, and never once land a hit or take another step closer — indistinguishable from a stall, since nothing about it looked wrong except that the fight never ended. Both states now share the tighter of the two, so "arrived" always means "can land."

Other players in the same game see the same monster doing the same things.

## How it works, in plain words

Two types exist: the Corpse, and since E-2.25 the Creepy, the same creature you play, added with no code at all: one data asset that points at the Creepy soul and the shared brain, and one row in the stats table. That is the test of the "monster as data" claim.

A monster is described by a data asset, one per monster type. The asset says which **soul** the monster wears (the Corpse's own), which **brain** it runs, which move it attacks with, how far it notices you, how close it needs to be to attack, and how much health it has. Designers change any of that in the asset, not in code.

The monster's body and moves come from the same soul asset a player equips. That is deliberate: the soul that drops from a monster later (E-2.7) is literally the kit it used on you.

The brain is a small state machine with three states: **idle**, **chase**, **attack**. A sense step, run a couple of times a second rather than every frame, picks the nearest player within range and keeps them as the target until they get well out of range. The state machine's transitions read that sense step; the chase state uses the engine's path following; the attack state fires the move through the same server path a player's key press takes, so the swing looks and behaves exactly like a player's.

The brain runs only on the server. Your machine never decides what a monster does; it receives where the monster is and which move it played.

A player standing in a safe zone is invisible to the sense step: never picked, and dropped the moment they step in (GDD 6.1; see [Zones](zones.md)).

A monster is tied to where it spawned by a **leash** (E-2.110). If a chase carries it further from that spot than its leash length (a value on its data asset, three thousand units to start with), or if it ever finds itself standing inside a safe zone such as the town, it gives up its target and walks back to its spawn point, ignoring every player on the way; once it is back, outside any safe zone, it hunts again. This is what stops Zone1's Corpses from following a player through the open seam into the town and loitering there. A monster whose spawn point itself lies in a safe zone stays there and never attacks. The host log says `leash: <monster> walks home (...)` and `leash: <monster> is home`.

When a monster dies it may leave a gem on the ground (see [Gems](gems.md)); the killer also earns experience (see [Level and growth](progression.md)).

## Settled

- Monsters are data. A new monster type is a new asset, not new code, once its soul exists.
- The brain is server-only and can be switched off for testing (`Slime.Monster.Brain 0`).
- Ranges, scan interval and despawn delay are rows on the asset. Health, mana, defense and resistances are the monster's row in the `DT_MonsterStats` table, keyed by the asset's name, so balance is a spreadsheet edit ([Combat](combat.md)).

## Waiting on design

- **The AI system itself** (D-1.3). The code uses Unreal's StateTree as the memo recommends; if the decision goes elsewhere, the state machine is rewritten, the data is not.
- **Monsters on the player's rules** (D-2.4). Built as the memo recommends: same pawn, same stats, same abilities, differences as data. Not yet decided.
- **How long a body should stay.** Despawn and sink seconds are rows on the monster asset, placeholders like the rest of its stats.
- **Whether killed monsters come back.** The spawner refills a slot after a delay from its table row, or never at 0. No design rule says which; the column is a playtest placeholder (register row "monster refill").

## Difficulty scales (S-3.3, partial)

Two host-editable scales, `FieldMonsterDifficulty` and `BossDifficulty` under
`[/Script/SandboxARPG.SandboxServerRules]`, read once at a monster's `BeginPlay`: the row's
`MaxHealth` and `DamageScale` are multiplied by whichever scale the row's Tier reads (Normal and
Elite read `FieldMonsterDifficulty`, Boss reads `BossDifficulty`; a boss-tier row multiplying its
own separate scale is D-2.4's rule that "asymmetry is a column, never a branch" — the read sits on
the data, not on an `IsA<...>` check). Health is scaled once, at spawn, so a host who changes the
rule mid-session never rescales an already-spawned monster; damage stays live because
`GetOutgoingDamageScale()` is read fresh on every hit. Both default to `1.0`, which is the row as
tuned in the playtests. `FieldMonsterDifficulty=2.0` doubles a Corpse's outgoing hit and its spawn
health; `BossDifficulty` is wired the same way but inert until a Boss-tier monster exists.

**Not built here (deliberately left out):** moveset selection per difficulty (GDD 8.1's
`BossMoveset` / `FieldMonsterMoveset` rows) — "how movesets vary" is OPEN and blocked on D-8.3. Mob
density and drop rates already scaled live before this row (`MobDensity`, `DropRates`, above and in
[Combat](combat.md)); this row only adds their two test cases and does not change their code.

## Where monsters come from

Since E-2.9a a level carries **spawners**: an invisible actor placed in the field that names a row of the `DT_SpawnTable` table. The row says which monster, how many, and how long an empty slot waits before it is filled again. When the level starts, the server spawns that many monsters at random walkable points within the spawner's radius; the host's **Mob density** rule multiplies the count (a row of 4 at density 2.0 is 8, at 0.5 it is 2). Kill one and, if the row's refill is above zero, another appears at a new point after the delay. Every client sees the monsters the way it sees any monster; the spawner itself is server-only and never sent over the network.

## For testers

Place a `MonsterSpawner` in a level, set its Spawn Row to `Forest_Corpse` or `Forest_Creepy` (4 Corpses or 2 Creepies, refill 30 s), Build Paths, play as a listen server. The Output Log line `MonsterSpawner ...: spawned N of M` says what happened; `refill in 30.0 s` follows a kill and `refilled` the return. Change `MobDensity` under `[/Script/SandboxARPG.SandboxServerRules]` in `Saved/Config/WindowsEditor/Game.ini` to see the count scale.

`Slime.SpawnMonster DA_Monster_Corpse` or `Slime.SpawnMonster DA_Monster_Creepy` (optionally a count) spawns in front of you, host only. `Slime.SetAttribute Health 0 FieldMonster_0` kills the first one. `Slime.Monster.Brain 0` before spawning leaves monsters standing.

## For engineers

`Source/SandboxARPG/Monsters/`: `MonsterDefinition.*` (the asset, Asset Manager type `Monster` under `/Game/Monsters`), `MonsterStats.*` (`FMonsterStatRow` and the `DT_MonsterStats` lookup by asset name, E-2.26), `FieldMonster.*` (the pawn, a subclass of the player pawn), `MonsterAIController.*` (StateTree component, started on possession), `MonsterStateTree.*` (the Monster Sense evaluator and Monster Attack task the tree asset wires; the tree recipe is in the header comment; the E-2.110 leash is `TickLeash` inside the evaluator: it reads `AFieldMonster::GetHomeLocation` (set at BeginPlay on the authority), `UMonsterDefinition::LeashRange` or the `Slime.Monster.Leash` cvar, and `IsInSafeZone`, drops the target so the tree falls to Idle, then drives `AAIController::MoveToLocation` home at the scan cadence while path following is idle; no tree asset edit), `MonsterConsoleCommands.cpp`, `SpawnTable.*` (`FSpawnRow` and the `DT_SpawnTable` lookup, config path in `DefaultGame.ini`), `MonsterSpawner.*` (the placed actor: BeginPlay spawns on the authority through `USandboxServerRules::Get().MobDensity`, contract C-2, at `UNavigationSystemV1::GetRandomReachablePointInRadius` points; `OnDestroyed` of each monster starts a refill timer). The wrapper ability component gained `ConfigureForAIPawn`, `InitVitals` and `OnHealthDepleted`. Death (E-2.78): `USoulComponent` ragdolls a body with no death clip but a physics asset (`Ragdoll` collision profile, `SetSimulatePhysics`, `bRagdolled`); `AFieldMonster` starts a sink `SinkSeconds` (asset row, default 1) before `DespawnDelay` ends, a 0.05 s timer that lowers the body one capsule height in total, physics off first, on every machine from the replicated dead flag, so the client copy finishes sinking as the server destroys it. Targeting (E-2.75): the sense evaluator in `MonsterStateTree.cpp` skips `IsDead()` pawns in `FindNearestPlayerPawn` and drops a target that died; the stall diagnostic logs `stalled 3 s with target ... self on navmesh %d, target on navmesh %d, navmesh building %d` once per stall through `UNavigationSystemV1::ProjectPointToNavigation`. The Corpse's kit lives in `Abilities/Corpse/` (E-2.48; see [Abilities](abilities.md)); `DA_Monster_Corpse` names Grave Clutch as its `AttackAbility`. The tree asset is `ST_Monster_Basic`, shared by every monster type. Difficulty scaling (S-3.3): `AFieldMonster::DifficultyScaleFor(Tier, FieldMonsterDifficulty, BossDifficulty)` (`FieldMonster.h`) is the pure read, exercised with no world by `Source/SandboxARPG/Server/Tests/DifficultyScaleTest.cpp` (`SandboxARPG.Server.Difficulty.*`); `AFieldMonster::BeginPlay` (`FieldMonster.cpp`) is the only caller, applied to `InitVitals`'s `MaxHealth` argument and to the stored `DamageScale`. Provisional markers: `docs/provisional.md` rows D-1.3, D-2.4, DS-1.1, "Corpse move name" and "monster refill". Design detail: `docs/decisions/D-1.3.md`, `D-2.4.md`, `DS-1.1.md`.
