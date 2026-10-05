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

A monster is tied to where it spawned by a **leash** (E-2.110). If a chase carries it further from that spot than its leash length (a value on its data asset, six thousand units since 2026-09-26, three thousand before; a target that gets more than four thousand units away is dropped, two thousand two hundred before, both raised because kiting past the old numbers ended a chase), or if it ever finds itself standing inside a safe zone such as the town, it gives up its target and walks back to its spawn point, ignoring every player on the way; once it is back, outside any safe zone, it hunts again. This is what stops Zone1's Corpses from following a player through the open seam into the town and loitering there. A monster whose spawn point itself lies in a safe zone stays there and never attacks. The host log says `leash: <monster> walks home (...)` and `leash: <monster> is home`.

A monster has two distances of its own (E-2.156, from the 2026-09-28 playtest). **Detection** (the asset's aggro range, nine hundred units since 2026-09-29, fifteen hundred before) is how close you must come before an idle monster notices you; at the default camera that is inside the screen's side edge, where fifteen hundred was past it, so monsters used to wake before you could see them. **Chase** (the lose-target distance, four thousand) is how far it follows once it has you, so stepping back and forth across the detection edge never breaks a chase. When a chase ends and nobody else is in detection, because you ran past the chase distance, died, left or stepped into a safe zone, the monster walks home the same way it does on the leash (`walks home (target lost; ...)`) instead of standing where it lost you. The leash is the outer bound from its spawn point, as above.

A monster that dies in the air (thrown by a knockback, or off a ledge) falls to the ground before it settles, rather than hanging where it died (E-2.154).

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

No monster spawns where players arrive. Every teleport destination (a gate's landing spot, a respawn point, a checkpoint well) is an arrival spot, and a spawn or a refill keeps the monster's aggro range plus a 300 uu margin away from all of them, so a player who teleports in is never seen by a monster standing there. The spot is not safe ground: a monster chasing you can still follow you to a well. If a spawner sits so close to a destination that no point in its radius is far enough, it takes the furthest point it found and warns once in the log, naming itself; move that spawner. The margin and the number of tries are `ArrivalKeepOutMargin` and `ArrivalKeepOutTries` under `[/Script/SandboxARPG.MonsterSpawner]` in `DefaultGame.ini`.

**No refill under a player (E-2.173).** A spawner brings nothing back while any player stands within 3,000 uu of it (the `RefillPlayerClearRange` row of `DT_CombatRules`). The refill time on its spawn row counts only while nobody is that near; when it has run out, every missing monster of the spawner returns together. So a pack you have killed stays dead while you fight on past it, and a zone a group has left is whole again for the next. The spawner looks every two seconds, and only while it has losses. The host log says `the pack returns N s after the last player leaves` at a death and `refilled N ... together` at the return. `Slime.SpawnerRefillGate 0` puts back the old rule, each slot on its own timer whoever is standing there. Provisional on the D-2.19 memo.

**Who lives where.** Levels rise by about two a zone, and each lair holds one Elite:

| Place | Monsters (level) | Lair |
|---|---|---|
| Starting forest | Corpse, the lone one (1) and the pack (2) | none |
| Zone 1, the Beach | Creepy (2), Corpse (3) | Corpse Elite (4) |
| Zone 2, the Fields | Gargoyle (4), Vampire (5), Ogre (5) | Vampire Elite (6) |
| Zone 3, the Jungle | Eloko (6), Treant (7) | Treant Elite (8) |
| Zone 4, the Pass | Vetala (8), Golem (9), Oni (9) | Golem Elite (10) |

**How thick.** In Zones 1 to 4 the packs stand beside the path, placed by `scripts/editor/populate_zones.py` (pythonscript commandlet, editor closed; `POPULATE_ZONES_DRY=1` prints the plan and writes nothing). A pack is three monsters of two types: two of the zone's first type and one of its second (on the Beach one Corpse and two Creepies, since a Corpse hits harder). Packs stand 2,600 uu of path apart in Zones 1 to 3 and 3,200 in Zone 4, further than a monster's 900 uu detection plus both packs' reach, so a fight held in place does not wake the next pack. A killed pack monster returns after 180 s and an Elite after 300 s. Every fourth pack of Zones 2 to 4 has an Elite with it; the Beach has its Elite in the lair only. That is 9, 13, 19 and 39 field monsters in Zones 1 to 4. The numbers were 6 a pack, 2,000 uu and 30 s until the evening of 2026-10-05, when a level 10 player died on the Beach under twelve monsters of two packs that kept refilling. They are sized for one player of the zone's level; a group raises the host's Mob density rule. A chase still runs 4,000 uu, so a player who flees through the zone gathers it. Spacing and Elite frequency per zone are `TUNE` at the top of the script; pack size and refill time are the `Zone<N>_Pack_*` rows of `Data/DT_SpawnTable.csv`, which a real run writes into the table asset.

The list of who stands in which field is the `"fields"` block of `Data/Authoring/Souls/monsters.json`: per zone a first and a second monster, in some a third with the spots it takes, and for each of a spawner's two slots (A and B) the monsters that may fill it. `scripts/editor/soul_monsters.py`, run by the pythonscript commandlet with the editor closed, re-points the placed spawners by slot from that block, so a field is repopulated by editing the JSON and running the script, not by moving spawners by hand. Placements and counts are stand-ins until the fields are designed.

A part of the map that streams out of the world and back in keeps its monsters (E-2.165): the spawner counts the ones still standing and only fills the gaps, where it used to spawn a whole new batch beside them. The host log says `kept N from before the cell left the world`.

## For testers

Place a `MonsterSpawner` in a level, set its Spawn Row to `Forest_Corpse` or `Forest_Creepy` (4 Corpses or 2 Creepies, refill 30 s), or one of the ten `Beach_*` rows of the beach blockout (CT-2.27: 2 to 4 each, the Corpse standing in for the beach's signature monster until it exists), Build Paths, play as a listen server. The Output Log line `MonsterSpawner ...: spawned N of M` says what happened; `refill in 30.0 s` follows a kill and `refilled` the return. Change `MobDensity` under `[/Script/SandboxARPG.SandboxServerRules]` in `Saved/Config/WindowsEditor/Game.ini` to see the count scale.

`Slime.SpawnMonster DA_Monster_Corpse` or `Slime.SpawnMonster DA_Monster_Creepy` (optionally a count) spawns in front of you, host only. `Slime.SetAttribute Health 0 FieldMonster_0` kills the first one. `Slime.Monster.Brain 0` before spawning leaves monsters standing.

## For engineers

`Source/SandboxARPG/Monsters/`: `MonsterDefinition.*` (the asset, Asset Manager type `Monster` under `/Game/Monsters`), `MonsterStats.*` (`FMonsterStatRow` and the `DT_MonsterStats` lookup by asset name, E-2.26), `FieldMonster.*` (the pawn, a subclass of the player pawn), `MonsterAIController.*` (StateTree component, started on possession), `MonsterStateTree.*` (the Monster Sense evaluator and Monster Attack task the tree asset wires; the tree recipe is in the header comment; the E-2.110 leash is `TickLeash` inside the evaluator: it reads `AFieldMonster::GetHomeLocation` (set at BeginPlay on the authority), `UMonsterDefinition::LeashRange` or the `Slime.Monster.Leash` cvar, and `IsInSafeZone`, drops the target so the tree falls to Idle, then drives `AAIController::MoveToLocation` home at the scan cadence while path following is idle; no tree asset edit), `MonsterConsoleCommands.cpp`, `SpawnTable.*` (`FSpawnRow` and the `DT_SpawnTable` lookup, config path in `DefaultGame.ini`), `MonsterSpawner.*` (the placed actor: BeginPlay spawns on the authority through `USandboxServerRules::Get().MobDensity`, contract C-2, at `UNavigationSystemV1::GetRandomReachablePointInRadius` points; `OnDestroyed` of each monster starts a refill timer; `PickSpawnPoint` keeps `AggroRange + ArrivalKeepOutMargin` from every `ATeleportDestination` read at BeginPlay, through the pure `DistanceToNearestArrival`, tested by `Monsters/Tests/SpawnerKeepOutTest.cpp`). The wrapper ability component gained `ConfigureForAIPawn`, `InitVitals` and `OnHealthDepleted`. Death (E-2.78): `USoulComponent` ragdolls a body with no death clip but a physics asset (`Ragdoll` collision profile, `SetSimulatePhysics`, `bRagdolled`); `AFieldMonster` starts a sink `SinkSeconds` (asset row, default 1) before `DespawnDelay` ends, a 0.05 s timer that lowers the body one capsule height in total, physics off first, on every machine from the replicated dead flag, so the client copy finishes sinking as the server destroys it. Targeting (E-2.75): the sense evaluator in `MonsterStateTree.cpp` skips `IsDead()` pawns in `FindNearestPlayerPawn` and drops a target that died; the stall diagnostic logs `stalled 3 s with target ... self on navmesh %d, target on navmesh %d, navmesh building %d` once per stall through `UNavigationSystemV1::ProjectPointToNavigation`. The Corpse's kit lives in `Abilities/Corpse/` (E-2.48; see [Abilities](abilities.md)); `DA_Monster_Corpse` names Grave Clutch as its `AttackAbility`. The tree asset is `ST_Monster_Basic`, shared by every monster type. Difficulty scaling (S-3.3): `AFieldMonster::DifficultyScaleFor(Tier, FieldMonsterDifficulty, BossDifficulty)` (`FieldMonster.h`) is the pure read, exercised with no world by `Source/SandboxARPG/Server/Tests/DifficultyScaleTest.cpp` (`SandboxARPG.Server.Difficulty.*`); `AFieldMonster::BeginPlay` (`FieldMonster.cpp`) is the only caller, applied to `InitVitals`'s `MaxHealth` argument and to the stored `DamageScale`. Provisional markers: `docs/provisional.md` rows D-1.3, D-2.4, DS-1.1, "Corpse move name" and "monster refill". Design detail: `docs/decisions/D-1.3.md`, `D-2.4.md`, `DS-1.1.md`.

## The level curve (E-2.169)

A monster's level can set its numbers. One table, `DT_MonsterLevels`, says what each level from 1 to 60 multiplies: life grows 10% a level up to level 20 and 7% after, damage grows 5% a level, and a kill pays 10 + 5 x level experience. Two more rows say what an Elite and a Boss multiply on top (an Elite has three times the life, 1.3 times the damage and pays three times the experience).

A monster type opts in by giving its row a life and a damage scale for level 1. The game then works out its real numbers from its level. A row without those two numbers is used exactly as typed. Every monster type that stands in Zones 1 to 4 is on the curve: its life, damage and experience come from its level-1 base and its `Level` in `DT_MonsterStats` through `DT_MonsterLevels`, with the Elite row on top for a lair's Elite (a Golem has a base of 120 life, so 257 at level 9, and its Elite 849 at level 10). The two forest Corpses and the plain `DA_Monster_Corpse` have no base and keep the numbers typed on their rows.

For testers: `python scripts/monster_ttk.py` prints, for every monster row, how many hits an on-level player needs to kill it and how many of its swings kill the player. `Slime.MonsterLevelCurve 0` takes every row as typed. A monster on the curve logs `level N on the curve: life ..., damage scale ..., ... XP` when it spawns.

For engineers: `UMonsterLevels` (`Monsters/MonsterLevels.h`): `Evaluate` is the arithmetic, `Resolve` reads the table; `AFieldMonster` calls it at spawn and when it pays experience. `scripts/gen_monster_levels.py` owns the formula and writes `Data/DT_MonsterLevels.csv` (`--check` compares). Every number is a placeholder.
