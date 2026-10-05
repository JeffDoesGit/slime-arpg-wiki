# Kit minions and monsters

**Status:** first pass, provisional (register row `soul webs`). Minions a v2 soul raises or takes over, the AI-control verbs (taunt, lure, entrance), how a field monster picks between several moves, and the ten souls as field monsters. The minion rules are "minion rules v0" of the Research Bank's 2026-10-03 minions study, which asks for its own design item; every number is a placeholder. Part of [Soul webs](soul-webs.md); the monsters themselves are [Field monsters](field-monsters.md).

## Minions

A minion is an ordinary field monster with an **owner**. There is no separate minion class.

- **Side.** A minion answers "hostile to X" as its owner would, and never against its owner or its owner's other minions. A wild monster is hostile to player pawns and to minions, never to another wild monster. The owner replicates, so every machine's plates, rings and cursor agree with the server.
- **Budget.** The `MinionBudget` rule row (6: "a minion budget of 6 points" in the minions study and in the Corpse, Treant and Eloko drafts; it was 4 until 2026-10-04) plus the owner's `MinionBudget` attribute, counted in **points**. Each minion carries its own cost: 1 unless its summon says otherwise (the move column `SummonCost`, the `Cost` param of `Summon`, `TakeOver` and `Sprout`, `SummonCost` on `SpendPlaced`; a Bewitch Thrall carries its 2 or 3), so half-point Saplings stand 12 at a budget of 6 and 16 at 8. A summon raises as many as the whole budget holds and the owner's oldest give way until they fit. The number is read from `Content/Data/DT_CombatRules`, which `scripts/editor/soul_monsters.py` refills from `Data/DT_CombatRules.csv`.
- **Life.** Timed (`SummonLife`) or until killed (0).
- **Revive.** A minion whose summon carried revive seconds (the move column `SummonRevive`, the param `Revive` on `Summon` and `Sprout`; the Treant's Saplings return after 8 s) is booked to return when it dies: by a hit, its timer, a sacrifice, a detonation or an order, never a dismissal. It comes back beside its owner with the life, cost and revive its summon asked for, if its owner still lives under the same soul outside a safe zone and the budget has room: a revive pushes nobody out, so a cast that filled the dead minion's place has taken it. While the owner's web hears the death, the op `ReviveMinion` changes the return (`Seconds` instead of its own time when sooner, `Sooner`, arg `AtDeath`, `AtAim` or `Never`; never under the `MinionReviveMinSeconds` row, 0.5) and the Treant's `ReviveFromSeed` spends the nearest Seed and brings the minion back there. `OnMinionRevived` is told when it has come up. `Slime.Minion.Show` prints how many are booked.
- **Mode.** The `MinionMode` op plants a minion (`Planted`): it is rooted where it stands, takes and keeps only targets within its reach (`MinionPlantedRange`, 700, edge to edge), throws at them for `MinionPlantedShare` (0.7) of its first move's damage once per `MinionRangedInterval` (1 s), does not follow and holds the op's modifiers (the draft's 30% less damage taken) until `Walking` uproots it. `Ranged` makes a minion that never closes in: it walks into its `Range` and throws for `Share`. The throw is the minion's own hit and lands at once; no thorn flies and nothing marks a planted minion on screen yet. Conditions `TargetIsPlanted`, `EventMinionIsPlanted`, `TargetNearOwnPlanted` read the mode; `Slime.Minion.Show` prints it.
- **Why it died.** `OnMinionDied` carries the cause as its noun: `Killed`, `Expired`, `Sacrificed`, `Detonated`, `Ordered` (eaten, pruned). The condition `EventNameIs` reads it. `Slime.Minion.Kill [Cause] [Count]` kills the oldest minions with a cause, for tests.
- **Stats.** A summoned minion takes its life and damage scale from the monster level curve at its **owner's** level. Its life is multiplied by 1 plus the owner's `MinionHealthIncreased` line (never under a tenth of the row); its damage scale by 1 plus the owner's `MinionDamageIncreased`, read off the owner at each hit. The host's monster difficulty rule is left out. A minion pays no mana.
- **What it fights.** The target its owner ordered first; else the nearest hostile within `MinionAggroRange` (900) of the **owner**, kept while it stays within `MinionLeashRange` (1500) of the owner. With nothing to fight it follows to within `MinionFollowDistance` (250). Past `MinionTeleportRange` (2500) it is placed beside the owner.
- **Credit.** A minion's kill credits its owner for XP, fragments, gem drops and flask charges, and does **not** fire the owner's on-kill nodes: the owner's web hears `OnMinionKill` instead. A minion dying credits nobody (no XP, no drops, no fragments); its owner's web hears `OnMinionDied`.
- **When they go.** No minion is raised in a safe zone. Every minion is dismissed, with no payoff and no event, when its owner dies, leaves, changes soul or steps into a safe zone, or when the minion itself stands in one. Never saved.

**Verbs.** A move row raises minions with `SummonName`, `SummonCount`, `SummonLife`, `SummonCost`, `SummonRevive`, and round the aim with `bSummonAtAim` (held to `SummonRange` of the user; the `Summon` op says `"arg": "Aim"` and `Range`); the name is a `DT_MonsterStats` row with or without its `DA_Monster_` prefix. From nodes:

| Op | What it does |
|---|---|
| `Summon` | Raises `Count` minions of a row round a spot, within the budget. Under `OnKill` it is a raise from a kill (a minion's own kill never fires it). |
| `TakeOver` | A live non-Boss wild monster becomes the user's minion for `Duration` (0 until killed), at most `Count` taken at once. It leaves its spawner's count as if it had died, so the slot refills on the row's timer; its kill is paid to the user once, at the take-over. |
| `CommandMinions` | `Attack`: every minion fights the target for `MinionCommandSeconds` (6). `Return`: for `MinionReturnSeconds` (2) they take no target and walk back. `Leap`: each is set down by the aim (no arc is played) and lands `Share` of its first move's damage within `MinionLeapRadius` (160). |
| `DetonateMinions` | Each minion bursts and dies; the hit is the minion's and Triggered. |
| `SacrificeMinion` | One minion dies; the user gains health and mana. |
| `MinionMode` | `Planted`, `Walking`, `Ranged`: see Mode above. |
| `ReviveMinion`, `ReviveFromSeed` | Under `OnMinionDied`: when and where the dead minion returns. |
| `CommandMinions` `Move` | Every minion, walking or planted, comes up round the aim after `Delay` seconds, mode kept; `OnMinionArrived` is told for each. |
| `CullMinions` | The owner cuts down its own minions (under a share of their life, or the oldest); the deaths are `Ordered`; the time left can be shared among the rest. |
| `TendMinions` | Seconds on a timed minion's life and a heal. |
| `TimedModifiers`, `Restore` on the `OwnMinions` target | Buff and heal. |

Events on the owner's web: `OnMinionSummoned`, `OnMinionHit`, `OnMinionKill`, `OnMinionDied`, `OnMinionRevived`, `OnMinionArrived`. The rule set `MinionRules` raises a minion's own hit by conditions on what it hits. `RedirectDamage` aimed at `OwnMinions` moves a share of a hit the owner takes to its minions (capped by `RedirectCap`, 0.4).

**The commanded move.** A command that names one move (Bewitch's command press; any order verb may do the same) marks the minion with that move's row before the move is asked for. The hits of that move are then the commanded move's, and the condition `EventMinionHitIsCommanded` holds for them under `MinionRules` and `OnMinionHit` (Fortissimo's more damage, Da Capo's less). The mark ends when the minion begins its next move, so a later swing of the same move that the minion picks for itself is not commanded, and a slow projectile that lands after the minion has begun another move has lost the mark. A command to a minion that cannot fire now (it is in the middle of a move, stunned, or the move is not ready) waits and fires when the wait is over. One command waits per minion and the newest wins; a minion that dies, or whose owner is gone, drops it.

**Reach and nearest.** An op aimed at `OwnMinions`, and `RedirectDamage` to `OwnMinions`, may carry `WithinReach`: only the minions within that many uu of the user, centre to centre in the ground plane, are acted on. With `NearestOnly` 1 the nearest of those alone; when none is in reach nothing happens, and a redirect moves nothing. The reach is measured from the user only, never from the aim or a placed thing.

**A clock per minion.** The condition `OncePerMinionSeconds` (seconds per rank) gives an op one clock for each minion the event is about, so Bell Collar adds a Toll at most once per interval for each Thrall, however many Thralls are hitting. It uses the op's per-target stamps, so an op carries it or `OncePerTargetSeconds`, never both.

Two minion types exist as data, with the numbers of their drafts since 2026-10-05: `DA_Monster_Bonewalker` (level-1 base 78 life, "a Normal field monster row", and damage scale 1.3333 on the Corpse's 6 Physical claw, the draft's 8; body scale 0.8) and `DA_Monster_Sapling` (base 27.3, "35% of a Normal monster's", and 0.4444 on the Treant's 9 Physical sweep, the draft's 4; body scale 0.25; `HitShapeScale` 0.5, so the sweep reaches 150 uu; picked once a second); both pay no XP and no fragments. At level 50 a Sapling has 1271 life and hits for 43.68, a Bonewalker 3631 and 87.37. `HitShapeScale` is a field of the monster definition (`"hitShapeScale"` in `monsters.json`) that multiplies the footprints of the monster's swings and its attack range. Not from the drafts yet: the share of the owner's flat damage (the `MinionRules` rule `OwnerFlatShare`, data), and the Sapling still sweeps a cone where the draft has one hit. Eloko's Thralls are taken-over monsters with rules of their own; see [the mechanics](kit-mechanics.md).

## AI control

Three verbs change what a field monster's brain does. They never act on a player-controlled pawn and never on a Boss-tier monster. The state lives on the monster and its sense evaluator reads it first.

- **Taunt** (a row's `TauntRadius` and `TauntSeconds`, or the `Taunt` op). Every field monster hostile to the user within the radius takes the user as its target for the seconds, whatever its scan would pick. The user's web hears `OnTaunted` once per monster taken. A minion is not taunted by its own side.
- **Lure.** The same call with a decoy point: the monsters walk to the point and swing at it. This is how a placed thing with `TauntRadius` and `Health` draws monsters; the damage they do goes to `UKitPlacedSubsystem::DamagePlaced`.
- **Entrance** (a row's `EntranceSeconds`, or the `Entrance` op). The monster walks toward the user and does not attack; a hit taken breaks it. A damaging move carries it on each landed hit; a move with no damage takes the monsters inside its shape's radius at the use. When it ends the user's web hears `OnControlEndedOnTarget`. It is not one of the replicated controls, so no plate shows it.

`Root` and `Pull` are controls on the wrapper ASC and also do nothing to a Boss: a root holds a pawn in place and lets it act; a pull is one launch toward the instigator, never past it (a pull or a knockback that finds its target still in the air of an earlier one rises only the rest of the way to the `KnockbackHeight` row, so a pull every channel tick does not lift its target), and tells the instigator's web `OnControlApplied` (name `Pull`). The rule is about another pawn's control: a root or any control a pawn puts on itself (Curl) is always taken, a player's too, at its full length and outside the caps page's clocks.

A pawn made untargetable (`UntargetableSeconds`, the `Untargetable` op) refuses hits, and monsters and minions skip it when they pick and keep a target.

## The move picker

A monster definition may list several **moves** (`UMonsterDefinition::Moves`): an ability of its soul with a `Weight`, a `Cooldown` of its own (beside the move row's), and a range band `MinRange` to `MaxRange` in uu, edge to edge. An empty list keeps the single attack move as before.

- **How close it walks.** The longest reach among its ready moves, so a ranged move off cooldown is fired from range; with none ready, the shortest reach of all, so it closes in and waits.
- **Which move fires.** Among the ready moves whose band holds the target's distance, one by weight. When no band holds it, among the ready moves that reach it, so a target nearer than every `MinRange` is still attacked.
- **Reach.** A `MaxRange` of 0 reads the move's own row: a dash its distance, a projectile most of its flight, an aimed move its cast range, a swing its hit shape.

A monster ignores mana cost; its moves are paced by these cooldowns.

## The ten souls as field monsters

A field monster wearing a v2 soul fires the kit's moves through the same phases as a player ([move phases](kit-move-phases.md)). It buys no nodes: each web's Unlock row and the centre row act, nothing else. `Data/Authoring/Souls/monsters.json` holds thirteen picker kits (the ten souls, a two-move Corpse kit for the forest's first Corpses, Bonewalker and Sapling), says which definition uses which kit, and names two souls per field.

`Data/DT_MonsterStats.csv` gains a Normal and an Elite row for each of the eight new souls and the two minion rows:

| Monster | Life in the row | Defense | Level | Elite (life in the row, level, name) |
|---|---|---|---|---|
| Ogre | 153 | 20 | 6 | 555, 8, Ogre Brute |
| Gargoyle | 97 | 40 | 6 | 351, 8, Gargoyle Sentinel |
| Treant | 212 | 30 | 10 | 770, 12, Elder Treant |
| Eloko | 96 | 0 | 9 | 350, 11, Eloko Conductor |
| Vampire | 242 | 0 | 14 | 877, 16, Vampire Elder |
| Vetala | 157 | 0 | 13 | 570, 15, Vetala Riddler |
| Golem | 607 | 50 | 18 | 2202, 20, Golem Warden |
| Oni | 391 | 20 | 17 | 1418, 19, Oni Warlord |

Every one of these rows carries a level-1 base (`LevelBaseHealth`, `LevelBaseDamageScale`), so at spawn its life, damage scale and XP come from the monster level curve at its level ([Field monsters](field-monsters.md)), not from the typed life.

**The spawn table re-point.** `Data/DT_SpawnTable.csv` gains 32 rows named by zone, place and soul (`Zone1_Shore_Gargoyle`, `Zone2_Lair_TreantElite` ...). The commandlet `scripts/editor/soul_monsters.py` re-points each grey-box spawner of Zones 1 to 4 on `Lvl_Slice_WP` by its row's suffix: a `_Corpse` row to the field's first soul, a `_Creepy` row to the second, the lair's elite to the first soul's Elite, the lair's adds to the second. Counts, refill times, radii and positions are kept; the forest keeps its Corpse and Creepy.

| Zone | Field | First soul | Second soul |
|---|---|---|---|
| Zone 1 | Beach | Ogre | Gargoyle |
| Zone 2 | Fields | Treant | Eloko |
| Zone 3 | Jungle | Vampire | Vetala |
| Zone 4 | Pass | Golem | Oni |

The assets and the map change only when the commandlet has been run under lock; see [the pipeline](soul-kit-pipeline.md).

## For testers

`Slime.Minion.Show` prints the local player's minions and the points they take of the budget. `Slime.Minion.Op <Op> [Target=..] [Name=..] [Arg=..] [Param=number ...]` runs one registered op as a rank-1 node would, with the nearest enemy as the event's pawn, for example `Slime.Minion.Op Summon Name=Bonewalker Count=2`. `Slime.SpawnMonster DA_Monster_Ogre` spawns a field monster of a new type. Authority only.

Log lines: `Minion: <owner> raises 2 DA_Monster_Bonewalker (2 serving, 2.0 of 6 point(s), life ...)`, `Minion: <minion> (<definition>) serves <owner> at level <n>: life <n>, damage scale <n>`, `Minion: <owner> asked for 20 <row> at 0.5 point(s) each with a budget of 6; 12 raised`, `Minion: <monster> is boss tier; <owner> cannot take it over`, `Minion: <minion> of <owner> killed <victim>; the kill is credited to <owner>`, `Minion: <owner> orders 2 minion(s) onto <enemy>`, `Minion: <owner> is in a safe zone; no <row> is raised`, `Taunt: <pawn> taunts within 600 uu for 4.0 s: 3 monster(s) taken`, `Entrance: <pawn> entrances within 400 uu for 3.0 s: 2 monster(s) taken`.

## What is not built

- Minion stances (the study's R6 is built without them); a planted minion "plants itself beside its target" only in that it plants where it stands; no thorn projectile, no ground ring for a planted cluster, the mode does not replicate.
- Minion orders: move all to the aim after a delay, carry and plant a Seed, graft into one, kill on recall (Treant Transplant, Gleaners, Graft, Prune); Corpse One Big Friend, the share at a raise, Bone Tax.
- A leap arc for a commanded leap: the minion is set down.
- A minion crossing a distance line as an event (Short Leash), a verb that moves ailments from the user to a minion and "the healthiest minion" as a target (Scapegoat), a placed thing that absorbs a share (Soul Jar), a reach measured from the aim or from a placed thing.
- A plate for an entranced monster.
- The vocabulary marks `MinionDamageIncreased` as not built: the read exists (`AFieldMonster::GetOutgoingDamageScale`), but no minion landed a hit in the proof runs.
- A picker condition beyond weight, cooldown and range band (the review's finding asks for "cooldown, range band and condition"; no condition column exists).
- The picking rule itself has no memo behind it; it is this pass's own.
- Nothing here has been seen on screen by the team.

## For engineers

`Source/SandboxARPG/Kits/KitMinions.*` (`FKitMinions`: the book of who owns what, `Budget`, `Summon`, `TakeOver`, `DismissAll`, the events, and the pure rules `EvaluateBudget`, `EvaluateSummon`, `EvaluateHealth`, `EvaluateDamageScale`), `Kits/KitAIControl.cpp` (`KitSeams::Taunt`, `Entrance`, `EntranceOne`), `Kits/Mechanics/MinionVerbs.cpp` (`MinionRules`, `TendMinions`, `SpendPlaced`). On the monster: `Monsters/FieldMonster.*` (the replicated `MinionOwner`, `InitAsMinion`, `BecomeMinion`, `DismissMinion`, `KillMinion`, `SetCommandTarget`, `SetCommandedMove`, `NoteMoveBegun`, `IsCommandedHit`, `QueueCommandedMove`, `OrderReturn`, `SetForcedTarget`, `SetLure`, `SetEntranced`, the picker's `GatherMoveOptions`, `PickMove`, `StampMoveUsed`), `Monsters/MonsterMovePicker.h` (`FMonsterMovePicker::ChaseRange` and `Pick`, pure), `Monsters/MonsterDefinition.h` (`FMonsterMove`, `Moves`, `BodyScale`), `Monsters/MonsterStateTree.*` (the sense evaluator reads the forced target, the lure, the entrance and the minion rules first; `MinionTakesTarget`, `MinionKeepsTarget`), `Monsters/MonsterSpawner.*` (`ReleaseMonster`). Tests: `SandboxARPG.Kits.Minions.*` (six), `SandboxARPG.Monsters.MovePicker.*` (three). Provisional markers: register row `soul webs`.

## A monster's own aura takes nothing from it (2026-10-04)

`SelfDamagePerSecond` on a toggle or a channel, and the Stoke toll on top of it, are paid by a player's pawn only (`FKitRules::PaysSelfDamage`). A field monster or a minion with a lit aura is paced by its move list and loses no health to it, as it pays no mana.

## Entrance options (2026-10-05)

A player whose web holds entrance rules changes how its entrances behave; with none, an entrance is as described above. The rules can slow or speed the entranced walk, add seconds for each stack of a state the monster holds, let it sleep through damage or wake at the first hit, make it count as Stunned for the `TargetHasControl` `Stun` condition, lock it out of a new entrance for some seconds after it ends, stop it at a set distance, and tell `OnEntranceArrived` once when it comes within a radius of where it walks. The `Entrance` op can send monsters to the user's nearest placed thing (arg `ToPlaced`) and scale the length (`Share`). The Eloko's Peal carries the base rule: 60% walk, +0.4 s per Toll, damage does not wake it, counts as Stunned, 4 s lockout, stands at 150 uu. The walk speed is a second scale on the monster's walk (`AMonsterCharacter::SetBrainWalkScale`), written on the authority and replicated by CharacterMovement. Data contract: schema section 7.14. Log lines: `Entrance: <monster> walks toward <user> and does not attack for 3.0 s, at 60% of its speed, damage does not wake it, counts as Stunned, stands at 150 uu`, `Entrance: <monster> arrives within 150 uu of <user>`, `Entrance: <monster> cannot be entranced for 3.3 s more (it woke from one)`. Not built: a plate or effect that shows the state.

## Bewitch's two presses (2026-10-05)

Bewitch's take-over takes the nearest monster to the aim that may be taken (it holds the Tolls and is of a tier the user may take), so a monster without Tolls standing nearer the cursor no longer turns the press into a command. The web hears `OnBewitchTakeOver` for a take-over and `OnBewitchCommand` for a command, so a node can answer one and not the other (schema 7.15). Log line: `Bewitch: <user>'s press was the command (N Thrall(s))`.
