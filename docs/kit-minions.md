# Kit minions and monsters

**Status:** first pass, provisional (register row `soul webs`). Minions a v2 soul raises or takes over, the AI-control verbs (taunt, lure, entrance), how a field monster picks between several moves, and the ten souls as field monsters. The minion rules are "minion rules v0" of the Research Bank's 2026-10-03 minions study, which asks for its own design item; every number is a placeholder. Part of [Soul webs](soul-webs.md); the monsters themselves are [Field monsters](field-monsters.md).

## Minions

A minion is an ordinary field monster with an **owner**. There is no separate minion class.

- **Side.** A minion answers "hostile to X" as its owner would, and never against its owner or its owner's other minions. A wild monster is hostile to player pawns and to minions, never to another wild monster. The owner replicates, so every machine's plates, rings and cursor agree with the server.
- **Budget.** The `MinionBudget` rule row (4) plus the owner's `MinionBudget` attribute, whole minions; every minion costs 1. A summon over budget replaces the owner's oldest.
- **Life.** Timed (`SummonLife`) or until killed (0). No revive.
- **Stats.** A summoned minion takes its life and damage scale from the monster level curve at its **owner's** level. Its life is multiplied by 1 plus the owner's `MinionHealthIncreased` line (never under a tenth of the row); its damage scale by 1 plus the owner's `MinionDamageIncreased`, read off the owner at each hit. The host's monster difficulty rule is left out. A minion pays no mana.
- **What it fights.** The target its owner ordered first; else the nearest hostile within `MinionAggroRange` (900) of the **owner**, kept while it stays within `MinionLeashRange` (1500) of the owner. With nothing to fight it follows to within `MinionFollowDistance` (250). Past `MinionTeleportRange` (2500) it is placed beside the owner.
- **Credit.** A minion's kill credits its owner for XP, fragments, gem drops and flask charges, and does **not** fire the owner's on-kill nodes: the owner's web hears `OnMinionKill` instead. A minion dying credits nobody (no XP, no drops, no fragments); its owner's web hears `OnMinionDied`.
- **When they go.** No minion is raised in a safe zone. Every minion is dismissed, with no payoff and no event, when its owner dies, leaves, changes soul or steps into a safe zone, or when the minion itself stands in one. Never saved.

**Verbs.** A move row raises minions with `SummonName`, `SummonCount`, `SummonLife`; the name is a `DT_MonsterStats` row with or without its `DA_Monster_` prefix. From nodes:

| Op | What it does |
|---|---|
| `Summon` | Raises `Count` minions of a row round a spot, within the budget. Under `OnKill` it is a raise from a kill (a minion's own kill never fires it). |
| `TakeOver` | A live non-Boss wild monster becomes the user's minion for `Duration` (0 until killed), at most `Count` taken at once. It leaves its spawner's count as if it had died, so the slot refills on the row's timer; its kill is paid to the user once, at the take-over. |
| `CommandMinions` | `Attack`: every minion fights the target for `MinionCommandSeconds` (6). `Return`: for `MinionReturnSeconds` (2) they take no target and walk back. `Leap`: each is set down by the aim (no arc is played) and lands `Share` of its first move's damage within `MinionLeapRadius` (160). |
| `DetonateMinions` | Each minion bursts and dies; the hit is the minion's and Triggered. |
| `SacrificeMinion` | One minion dies; the user gains health and mana. |
| `TendMinions` | Seconds on a timed minion's life and a heal. |
| `TimedModifiers`, `Restore` on the `OwnMinions` target | Buff and heal. |

Events on the owner's web: `OnMinionSummoned`, `OnMinionHit`, `OnMinionKill`, `OnMinionDied`. The rule set `MinionRules` raises a minion's own hit by conditions on what it hits. `RedirectDamage` aimed at `OwnMinions` moves a share of a hit the owner takes to its minions (capped by `RedirectCap`, 0.4).

Two minion types exist as data: `DA_Monster_Bonewalker` (level-1 base 30 life and damage scale 0.5, body scale 0.8) and `DA_Monster_Sapling` (base 12 and 0.25, body scale 0.25); both pay no XP and no fragments. Eloko's Thralls are taken-over monsters with rules of their own; see [the mechanics](kit-mechanics.md).

## AI control

Three verbs change what a field monster's brain does. They never act on a player-controlled pawn and never on a Boss-tier monster. The state lives on the monster and its sense evaluator reads it first.

- **Taunt** (a row's `TauntRadius` and `TauntSeconds`, or the `Taunt` op). Every field monster hostile to the user within the radius takes the user as its target for the seconds, whatever its scan would pick. The user's web hears `OnTaunted` once per monster taken. A minion is not taunted by its own side.
- **Lure.** The same call with a decoy point: the monsters walk to the point and swing at it. This is how a placed thing with `TauntRadius` and `Health` draws monsters; the damage they do goes to `UKitPlacedSubsystem::DamagePlaced`.
- **Entrance** (a row's `EntranceSeconds`, or the `Entrance` op). The monster walks toward the user and does not attack; a hit taken breaks it. A damaging move carries it on each landed hit; a move with no damage takes the monsters inside its shape's radius at the use. When it ends the user's web hears `OnControlEndedOnTarget`. It is not one of the replicated controls, so no plate shows it.

`Root` and `Pull` are controls on the wrapper ASC and also do nothing to a Boss: a root holds a pawn in place and lets it act; a pull is one launch toward the instigator, never past it.

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

`Slime.Minion.Show` prints the local player's minions and budget. `Slime.Minion.Op <Op> [Target=..] [Name=..] [Arg=..] [Param=number ...]` runs one registered op as a rank-1 node would, with the nearest enemy as the event's pawn, for example `Slime.Minion.Op Summon Name=Bonewalker Count=2`. `Slime.SpawnMonster DA_Monster_Ogre` spawns a field monster of a new type. Authority only.

Log lines: `Minion: <owner> raises 2 DA_Monster_Bonewalker (2 of 4 serving, life ...)`, `Minion: <minion> (<definition>) serves <owner> at level <n>: life <n>, damage scale <n>`, `Minion: <owner> asked for 3 <row> with a budget of 4; 2 raised`, `Minion: <monster> is boss tier; <owner> cannot take it over`, `Minion: <minion> of <owner> killed <victim>; the kill is credited to <owner>`, `Minion: <owner> orders 2 minion(s) onto <enemy>`, `Minion: <owner> is in a safe zone; no <row> is raised`, `Taunt: <pawn> taunts within 600 uu for 4.0 s: 3 monster(s) taken`, `Entrance: <pawn> entrances within 400 uu for 3.0 s: 2 monster(s) taken`.

## What is not built

- Minion stances (the study's R6 is built without them) and revive.
- A leap arc for a commanded leap: the minion is set down.
- A plate for an entranced monster.
- The vocabulary marks `MinionDamageIncreased` as not built: the read exists (`AFieldMonster::GetOutgoingDamageScale`), but no minion landed a hit in the proof runs.
- A picker condition beyond weight, cooldown and range band (the review's finding asks for "cooldown, range band and condition"; no condition column exists).
- The picking rule itself has no memo behind it; it is this pass's own.
- Nothing here has been seen on screen by the team.

## For engineers

`Source/SandboxARPG/Kits/KitMinions.*` (`FKitMinions`: the book of who owns what, `Budget`, `Summon`, `TakeOver`, `DismissAll`, the events, and the pure rules `EvaluateBudget`, `EvaluateSummon`, `EvaluateHealth`, `EvaluateDamageScale`), `Kits/KitAIControl.cpp` (`KitSeams::Taunt`, `Entrance`, `EntranceOne`), `Kits/Mechanics/MinionVerbs.cpp` (`MinionRules`, `TendMinions`, `SpendPlaced`). On the monster: `Monsters/FieldMonster.*` (the replicated `MinionOwner`, `InitAsMinion`, `BecomeMinion`, `DismissMinion`, `KillMinion`, `SetCommandTarget`, `OrderReturn`, `SetForcedTarget`, `SetLure`, `SetEntranced`, the picker's `GatherMoveOptions`, `PickMove`, `StampMoveUsed`), `Monsters/MonsterMovePicker.h` (`FMonsterMovePicker::ChaseRange` and `Pick`, pure), `Monsters/MonsterDefinition.h` (`FMonsterMove`, `Moves`, `BodyScale`), `Monsters/MonsterStateTree.*` (the sense evaluator reads the forced target, the lure, the entrance and the minion rules first; `MinionTakesTarget`, `MinionKeepsTarget`), `Monsters/MonsterSpawner.*` (`ReleaseMonster`). Tests: `SandboxARPG.Kits.Minions.*` (six), `SandboxARPG.Monsters.MovePicker.*` (three). Provisional markers: register row `soul webs`.
