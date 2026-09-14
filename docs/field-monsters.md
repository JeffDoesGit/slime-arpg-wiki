# Field monsters

*First monster built in E-2.5 (branch `feat/field-monster-creepy`). Provisional on two design calls: the AI system (D-1.3) and whether monsters play by the player's rules (D-2.4).*

## What you see

A sword-carrying skeleton (the Corpse, from the City of Brass pack) standing in the field, chosen because you play a Creepy and a monster that looks like you is hard to read. Walk toward it and, once you are close enough, it turns and comes for you along the ground, avoiding walls. When it reaches you it claws at you twice (Grave Clutch, its kit's first move since E-2.48; it carries no sword), waits for the swing to finish, and swings again. Walk far enough away and it gives up and stands still. It cannot hurt you yet, and you cannot hurt it: hits, damage and death animations are the next item (E-2.6). The one thing that kills it today is a developer command that sets its health to zero, after which it stops, becomes walk-through, and vanishes a few seconds later.

Other players in the same game see the same monster doing the same things.

## How it works, in plain words

Two types exist: the Corpse, and since E-2.25 the Creepy, the same creature you play, added with no code at all: one data asset that points at the Creepy soul and the shared brain, and one row in the stats table. That is the test of the "monster as data" claim.

A monster is described by a data asset, one per monster type. The asset says which **soul** the monster wears (the Corpse's own), which **brain** it runs, which move it attacks with, how far it notices you, how close it needs to be to attack, and how much health it has. Designers change any of that in the asset, not in code.

The monster's body and moves come from the same soul asset a player equips. That is deliberate: the soul that drops from a monster later (E-2.7) is literally the kit it used on you.

The brain is a small state machine with three states: **idle**, **chase**, **attack**. A sense step, run a couple of times a second rather than every frame, picks the nearest player within range and keeps them as the target until they get well out of range. The state machine's transitions read that sense step; the chase state uses the engine's path following; the attack state fires the move through the same server path a player's key press takes, so the swing looks and behaves exactly like a player's.

The brain runs only on the server. Your machine never decides what a monster does; it receives where the monster is and which move it played.

A player standing in a safe zone is invisible to the sense step: never picked, and dropped the moment they step in (GDD 6.1; see [Zones](zones.md)).

When a monster dies it may leave a gem on the ground (see [Gems](gems.md)); the killer also earns experience (see [Level and growth](progression.md)).

## Settled

- Monsters are data. A new monster type is a new asset, not new code, once its soul exists.
- The brain is server-only and can be switched off for testing (`Slime.Monster.Brain 0`).
- Ranges, scan interval and despawn delay are rows on the asset. Health, mana, defense and resistances are the monster's row in the `DT_MonsterStats` table, keyed by the asset's name, so balance is a spreadsheet edit ([Combat](combat.md)).

## Waiting on design

- **The AI system itself** (D-1.3). The code uses Unreal's StateTree as the memo recommends; if the decision goes elsewhere, the state machine is rewritten, the data is not.
- **Monsters on the player's rules** (D-2.4). Built as the memo recommends: same pawn, same stats, same abilities, differences as data. Not yet decided.
- **Death.** The Corpse has death clips, but nothing plays them yet; the monster just stops and disappears. Hit reactions and death are E-2.6.
- **Whether killed monsters come back.** The spawner refills a slot after a delay from its table row, or never at 0. No design rule says which; the column is a playtest placeholder (register row "monster refill").

## Where monsters come from

Since E-2.9a a level carries **spawners**: an invisible actor placed in the field that names a row of the `DT_SpawnTable` table. The row says which monster, how many, and how long an empty slot waits before it is filled again. When the level starts, the server spawns that many monsters at random walkable points within the spawner's radius; the host's **Mob density** rule multiplies the count (a row of 4 at density 2.0 is 8, at 0.5 it is 2). Kill one and, if the row's refill is above zero, another appears at a new point after the delay. Every client sees the monsters the way it sees any monster; the spawner itself is server-only and never sent over the network.

## For testers

Place a `MonsterSpawner` in a level, set its Spawn Row to `Forest_Corpse` or `Forest_Creepy` (4 Corpses or 2 Creepies, refill 30 s), Build Paths, play as a listen server. The Output Log line `MonsterSpawner ...: spawned N of M` says what happened; `refill in 30.0 s` follows a kill and `refilled` the return. Change `MobDensity` under `[/Script/SandboxARPG.SandboxServerRules]` in `Saved/Config/WindowsEditor/Game.ini` to see the count scale.

`Slime.SpawnMonster DA_Monster_Corpse` or `Slime.SpawnMonster DA_Monster_Creepy` (optionally a count) spawns in front of you, host only. `Slime.SetAttribute Health 0 FieldMonster_0` kills the first one. `Slime.Monster.Brain 0` before spawning leaves monsters standing.

## For engineers

`Source/SandboxARPG/Monsters/`: `MonsterDefinition.*` (the asset, Asset Manager type `Monster` under `/Game/Monsters`), `MonsterStats.*` (`FMonsterStatRow` and the `DT_MonsterStats` lookup by asset name, E-2.26), `FieldMonster.*` (the pawn, a subclass of the player pawn), `MonsterAIController.*` (StateTree component, started on possession), `MonsterStateTree.*` (the Monster Sense evaluator and Monster Attack task the tree asset wires; the tree recipe is in the header comment), `MonsterConsoleCommands.cpp`, `SpawnTable.*` (`FSpawnRow` and the `DT_SpawnTable` lookup, config path in `DefaultGame.ini`), `MonsterSpawner.*` (the placed actor: BeginPlay spawns on the authority through `USandboxServerRules::Get().MobDensity`, contract C-2, at `UNavigationSystemV1::GetRandomReachablePointInRadius` points; `OnDestroyed` of each monster starts a refill timer). The wrapper ability component gained `ConfigureForAIPawn`, `InitVitals` and `OnHealthDepleted`. The Corpse's kit lives in `Abilities/Corpse/` (E-2.48; see [Abilities](abilities.md)); `DA_Monster_Corpse` names Grave Clutch as its `AttackAbility`. The tree asset is `ST_Monster_Basic`, shared by every monster type. Provisional markers: `docs/provisional.md` rows D-1.3, D-2.4, "Corpse move name" and "monster refill". Design detail: `docs/decisions/D-1.3.md`, `D-2.4.md`.
