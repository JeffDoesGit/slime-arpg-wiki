# Soul webs

**Status:** first pass, provisional (register row `soul webs`). Built 2026-10-03 on Jon's word from the ten soul drafts; the contract is `docs/design/soul-kit-schema-v2.md`. Every number in every row is a placeholder. Nothing on this page has been seen on screen by the team yet. Related pages: [the pipeline](soul-kit-pipeline.md), [states and ailments](kit-states-and-ailments.md), [move phases](kit-move-phases.md), [placed things](kit-placed-things.md), [minions](kit-minions.md), [the ten mechanics](kit-mechanics.md).

## The idea

A soul of the second kit schema (a "v2 soul") has a basic attack, six abilities and two trees. The **active tree** is one web per ability: the web's root is an Unlock node that grants the ability, and every other node of that web changes that ability. The **passive tree** is one radial web of 39 slots on four rings round a free centre. Every node is ranked, and each rank costs one point.

Ten souls have webs: Creepy, Corpse, Ogre, Oni, Gargoyle, Golem, Eloko, Treant, Vampire, Vetala. A soul is a v2 soul when the table `DT_KitAbilities` has rows for it; nothing else marks it.

A v2 soul acts only while it is your **primary** soul. Both trees are live together; the secondary slot does nothing for a v2 soul. While a v2 soul is the primary, the older two-region tree ([Level and growth](progression.md)) has no soul in either slot.

## What you see

The skill tree key and its bar disc open the soul web screen in place of the old tree screen. There is a tab per ability web and one for the passive web. A small node is a circle, a notable a larger ringed circle, a defining node a rounded square, a keystone a diamond, and the Unlock node wears the ability's icon; each shows its rank as n/Max. The detail panel shows the node under the cursor: the text of its next rank, a keystone's price, and a line saying how much of that text the game enacts today (see "What is not built"). A left click picks one more rank, a right click drops one; Confirm sends every pick as one request, Cancel drops them. Respec turns the clicks into ranks to give back.

The ability bar has a seventh slot (key 5) so the basic attack and six abilities fit.

## Points

Two pools, each `level - 1` points (49 at level 50): **active** points for ability webs, **passive** points for the passive web. The pools are per soul: a point spent on one soul's webs is not gone for another's. No design source says which way that should go; this is the reading built. Points are never stored: what is left is the pool's size less the ranks held, so a give-back returns its points by derivation.

## Allocation rules

One function decides (`FKitRules::EvaluateAllocate`), asked by the screen for its colours and by the server for the answer. The checks run in this order and the first that fails gives the reason:

| Check | Refusal text |
|---|---|
| The row exists on this soul | "No such node" |
| The passive centre | "Every soul has it" |
| Rank below the node's `MaxRank` | "Already at its highest rank" |
| Active web, not the Unlock node: the web's Unlock node holds a rank | "Unlock the ability first" |
| Active web: the node's `Requires` node holds a rank | "Needs <node> first" |
| Active web: fewer than 12 ranks held in the web, the Unlock node counted | "This web is full (12 ranks)" |
| Active web, a Defining node's first rank: 2 ranks already held in its web | "Needs 2 points in this web first" |
| Passive, first rank: the ring is open (ring 2 at 4 passive points spent, ring 3 at 12, ring 4 at 20) | "Ring <n> opens at <m> passive points spent" |
| Passive, first rank: a neighbouring slot holds a rank (the centre always does) | "Needs a neighbour first" |
| Passive, first rank of a keystone: fewer than 2 keystones held | "At most 2 keystones" |
| First rank: no node of its `Excludes` list holds a rank | "Cannot go with <node>" |
| A point is left in the node's pool | "No active points left" / "No passive points left" |

A request is a list of rows and ranks. The server walks it in order, each rank seeing the earlier ones as bought, and buys **all of it or none**.

**Giving back** (`EvaluateGiveBackList`): the whole list is taken off together and what is left has to stand: every held node still has its Unlock and `Requires` node, a Defining node still has 2 other ranks in its web, every held passive slot still reaches the centre through held slots and its ring is still open on the remaining points. Refusals: "No such node", "Comes with the soul; it stays" (the centre), "Not taken", "Its other ranks still need it", "<node> still needs it; give that back first". Respec is free and sits behind the same switch as the old tree's (`bRespecButton` in `UHudLayoutSettings`; the web screen's button also needs `bWebRespecButton`). With the switch off the server logs `respec refused: respec is off`.

**A move that is locked** says why when pressed: "Locked: unlock <ability> in its web"; "A move of the old tree; this soul plays by its webs" (a legacy move left in the Creepy's or Corpse's moveset); "Not one of your moves"; with the webs off, "A move of the soul webs, which are off". The basic attack is always granted.

**Field monsters buy nothing.** A monster wearing a v2 soul counts every web's Unlock row as held (so the ability's own ops and overrides act), the centre row acts, and no other node does. It is never asked whether a move is unlocked.

## Session only

Allocations live in memory on the pawn and replicate; they are never handed to the character record. Nothing under `Server/Persistence` reads the component. The reason is CLAUDE.md 1.9: provisional code writes no persistent data, and the point economy, the caps and every node are provisional. A new session starts with no ranks.

## The switch

`Slime.SoulWebs` (default 1). At 1 a soul with rows in `DT_KitAbilities` plays by its webs. At 0 the legacy trees of the Creepy and the Corpse are back, every kit move is locked (but for the kit basic attack of a soul that has no legacy default attack), and none of the caps or late verbs act. Re-equip the soul after changing it.

## How a move's numbers are resolved

`USoulWebComponent::ResolveMove` builds the row a move is played with, on any machine, from replicated ranks:

1. The move's `DT_MoveStats` row.
2. The shape on the soul asset (`HitRange`, `HitArc`, `HitDelay`, `ProjectileRange`).
3. Every held node's `Overrides` for this ability at the node's rank: `Set`, `Add` or `Multiply` on a numeric, bool or enum column (an enum by its index). A value list shorter than the node's ranks repeats its last value. Steps 1 to 3 are cached until a rank, the worn soul or a table changes.
4. On the authority only: `OverrideRules` rules whose conditions hold now (a column changed while a condition holds; a client shows the row without it).
5. The live attribute lines: `AttackSpeed` or `CastSpeed` multiplies the play rate by the ability's kind (no ceiling: the swing's times follow it and only the clip on the mesh is clamped, see [move phases](kit-move-phases.md)), `CooldownRecovery` divides the cooldown, `ManaCostLess` cuts the cost (capped at 90 %), `AreaOfEffect` scales the six radii, `SkillDuration` the buff's time.

Two more overrides reach past the move's own row. A **row override** (`RowOverrides`) changes one field of a state, ailment or placed row for the node's owner; the owner's resolved copy is what its states, ailments and placed things use (a state on another pawn and an ailment take the applier's copy). An **op override** (`OpOverrides`) switches off or re-numbers one base op of an ability (carried on its Unlock row) or of the soul (carried on the centre row) while the overriding node holds a rank.

A node's `Modifiers` are attribute changes held as one set per node while it holds a rank and its soul is worn. An `Increased` or `More` line on an attribute that starts at 0 (every v2 attribute, and `MoveSpeed`, the crit lines, `Accuracy`, the resistances and penetrations) is applied as `Flat`, or it would multiply nothing. A negative line works on the signed fractions (`MoveSpeed`, `AttackSpeed`, `CastSpeed`, `DamageTakenLess`, the three damage-kind lines and the two minion lines go down to -1; see [Stats](attribute-set.md)): `MoveSpeed` -1 pins the user, `AttackSpeed` -0.3 plays an attack's clip at 70 %. On any other attribute a total under 0 reads as 0.

## The op dispatcher

A node's verbs are **ops**: when a trigger fires and every condition holds, the op acts on its target. All four are names looked up in registries (`FKitOps`), so a new one is added from its own `.cpp` with one registrar object. Server only.

- **Hits and touches.** `OnHit` is told for every landed hit of the user that is no ailment tick. A move that deals no damage still tells it: a kit row with no `FlatDamage` that names a shape (`ShapeRadius` or `ShapeLength`; Hypnotic Peal's circle) swings at its contact frame, and a kit dash with no damage (Mist Step) crosses, and each enemy reached is a **touch**: the web hears `OnHit` with that enemy and 0 damage, the row's own state, ailment, pull and control land, and nothing else happens to the enemy (no roll, no crit, no Health written, no flinch, no hit stop, no thorns, no hit-taken, never a kill). The log line is `Touch: <user>'s <ability> touches <enemy> (no damage)`. A swing's kind follows its ability's tags: melee only when tagged `Melee`, area when tagged `Area` or resolved at the aim point, both for a sweep tagged both; `HitIsMelee`, `HitIsArea`, the melee and area damage lines and the melee-only thorns read that. A legacy (non-kit) move's swing is melee as before.
- **Trigger.** The place an event happens calls `USoulWebComponent::Dispatch` with a context (the user, the other pawn, the hit and its outcome, the ability, the event's noun and amount, the aim, the place in the chain). An op of an ability web is limited to its own ability unless its row says otherwise; a trigger that takes a noun (a state, an ailment, a control kind, a placed row) filters on the op's `TriggerName`. 70 triggers are in the vocabulary.
- **Order.** The ops of a trigger run in row order, the ability's own (its Unlock row) first, then the nodes'. An op that says `before` runs ahead of the ones that do not, and for `OnHit` ahead of the move row's own plants as well (`USoulWebComponent::Dispatch` with its two halves; schema 7.9).
- **Conditions.** All must hold; conditions that share an any-of group need one of the group. `bNot` inverts one. A condition about the target pawn (`TargetHasState`, `TargetNotBoss` ...) on an op whose target names many pawns is read once per pawn, and the op acts on the pawns that pass, when the event has no pawn of its own or the condition says `each`; otherwise it is about the event's pawn (`FKitOps::ReadsPerPawn`, schema 7.9). A condition with no handler never holds (warned once). The ones that keep a stamp (`CooldownSeconds`, `OncePerTargetSeconds`) write it only when every condition of the op passed.
- **Op.** The handler runs once per target. Standing rules (`BonusDamage`, `ScaleByStat`, `DamageTaken`, `CheatDeath`, `WhileModifiers`) are never dispatched: they are read where they apply (when a hit is built, in `ApplyHit`, on the web's timer) whatever trigger the row names. A `ScaleByStat` whose target is no line of a hit (`Defense`, `Thorns`, `AilmentDamageIncreased`) is a bonus on that attribute that follows its source stat: the web's step works it out four times a second and writes it again only when it has moved (`FKitRules::DerivedStatBonus`).
- **The origin rule.** An op that makes a hit, a consumption that pays damage, or a chain asks `FKitOps::MayTrigger` and is refused under a Triggered or Expiry cause; an op that plants a state or an ailment asks `FKitOps::MayPlant`, which also lets a chain hop plant (the hop is the move's own hit carried on). With `Slime.Kit.Trace` a refusal reads `trace op <node> <op> refused (origin): ...`. An op with no handler does nothing and warns once.
- **Target.** `Self`, `HitTarget`, `Attacker`, `Aim`, `EnemiesAroundSelf`, `EnemiesAroundTarget`, `AlliesAroundSelf`, `OwnPlaced`, `OwnMinions`. With none, the event's pawn for a hit event and the user otherwise.

**The press budget.** The budget bounds what a press sets off past its own nodes. An op that answers the press's own events (its use, its hits, its kills, the states its moves plant) draws nothing, so every node of a full build runs. An op that runs inside another op's handler, answering what that op did (a reaction of a reaction), draws one of 12 (`FKitRules::DrawsOnBudget`), and **one op is one effect**: the budget counts the different ops that draw, not the pawns they act on or how often they are told. A thirteenth different op is refused with a Warning, once per press. The number is the `PressBudget` row of `Data/DT_CombatRules.csv`. A recast, a channel tick, an aura's round and a repeat each start a full budget; an event no press set off has 12 of its own.

**The lists stand while an op runs.** A rank bought, a level gained or a soul swapped from inside an op's handler (a kill that levels the user up) does not rebuild the op lists, the node modifiers or the resolved rows there and then: the rebuild is deferred until the outermost dispatch, rule read or hit being built has returned (`USoulWebComponent::FOpListScope`), so the op that is running, and the patched copy of it a context points at, stay valid. A dispatch that sees the lists were rebuilt under it ends.

**The origin rule.** The only recursion guard is the E-2.166 rule ([Combat](combat.md)): an op that makes a hit, a stack or an ailment marks it Triggered, one link deeper, and only a Direct cause below the depth bound sets one off. `OnKill` and the state and ailment events ask the depth bound only, since a kill by a tick or a cap reached by planted stacks is what those nodes are for.

**The timer.** No tick. One timer at four steps a second while a v2 soul is worn runs `WhileModifiers`, `OnInterval`, `OnStill`, `OnBuffEnd`, timed modifier sets running out and an armed cheat-death heal.

**An interval** (`OnInterval`, param `Seconds`) is "every N seconds while its conditions hold". The op's conditions are read ahead on every step (a peek: a `Chance` is not rolled, a cooldown is not stamped, the ones read per pawn wait for the run). The clock starts on the step they are first seen to hold and the first firing is N seconds later; it stops when they stop holding. The schedule carries its remainder, so an op fires every N seconds on average whatever the step (0.3 s is thirty firings in nine seconds), on the step nearest its due time and at most once a step. An op whose `Seconds` is no longer than the step (0.25) is a poll: it fires on every step its conditions hold, the first included. The param `Immediate` (1) asks any op for a firing on the step its conditions are first seen to hold.

## Where the tables live

Six tables under `/Game/Data/Kits/`, named in `Config/DefaultGame.ini` under `[/Script/SandboxARPG.KitTables]`: `DT_KitNodes` (968 rows: per soul 6 Unlock rows, its active nodes, 39 passive slots and one centre row; named `<Soul>.<Web>.<NodeId>`), `DT_KitAbilities` (70 rows, seven per soul), `DT_KitStates` (71), `DT_KitAilments` (6), `DT_KitPlaced` (28), `DT_KitCues` (24). The kit moves' numbers are rows of `DT_MoveStats`. All of them are generated from `Data/Authoring/Souls/` and filled by commandlet; see [the pipeline](soul-kit-pipeline.md). A missing table is warned about once and reads as empty.

## For testers

All commands are dev tooling, compiled out of Shipping; the ones that change something run on the host or in single player.

- **Webs.** `Slime.Web.Show [Pawn]` (points, each node's rank and whether one more may be bought); `Slime.Web.Allocate <Row> [Ranks] [Pawn]` (the full row name or its node id, through the rule the screen's Confirm reaches); `Slime.Web.Max <Web> [Pawn]` (every rank of one web the rules and points allow; the web is the ability's row name, or `Passive`); `Slime.Web.Respec [Pawn]`; `Slime.Web.Reload` (reads the tables again after a reimport); `Slime.Web.Rows <State|Ailment|Placed> <Row> [Pawn]` (a row as the pawn's nodes resolve it beside the table's own).
- **The screen.** `Slime.ToggleWebScreen`; `Slime.Web.Screen <tab <Web>|click <Node>|unclick <Node>|confirm|cancel|respec>` drives the open screen as the mouse would; `Slime.Web.DumpScreen [Pawn]` logs what each tab would show; `Slime.Hud.Kit` logs the bar's kit slots, the status row and the ground marks as this machine draws them.
- **States and ailments.** `Slime.State.Show [Pawn]`, `Slime.State.Add <Name> <Stacks> [Pawn]`, `Slime.Ailment.Show [Pawn]`.
- **Placed things.** `Slime.Placed.Show`, `Slime.Placed.Place <Row> [Distance] [Pawn]`, `Slime.Placed.Burst [Row] [Pawn]`.
- **Minions.** `Slime.Minion.Show`, `Slime.Minion.Op <Op> [Target=..] [Name=..] [Arg=..] [Param=number ...]`.
- **Phases.** `Slime.Phase.Show`, `Slime.ReleaseMove <AbilityClass>`.
- **Proof tooling.** `Slime.Kit.Op <Op|Test> ...` runs one op or tests its conditions as a rank-1 node would; `Slime.Kit.Control <kind> [Magnitude] [Duration] [Pawn|Self|From=<Pawn>]`; `Slime.Kit.Column <AbilityRow> <Column> <Value> | clear` sets a move column for this process only; `Slime.Kit.Probe <Trigger> <Kind>[:<Name>[:<Value>]] | clear` logs a condition at every telling of a trigger; `Slime.Kit.Late [Pawn]` prints the hard-control clock, immunities, channel ticks and hit counts; `Slime.Kit.RequestAllocate <full row name> [Ranks]` and `Slime.Kit.ClientUse <AbilityClass> [DistanceAhead]` go through the client's RPC from any machine; `Slime.Kit.ClientShow` prints what this machine holds of ranks, phase, charges, toggles, placed list and status views.
- **Others.** `Slime.Curse.Show` (Vetala's curses); `Slime.Cue <CueRow> [Pawn] [Times]` and the variable `Slime.Cue.Log 1`.

Log lines you will see:

- `SoulWeb: <pawn> allocated <row> rank <n>, ... (active points left N, passive points left M)`
- `SoulWeb: <pawn> refused <k> rank request(s) at <row>: <reason>`
- `SoulWeb: <pawn> gave back ...` and `SoulWeb: <pawn> respec refused at <row>: <reason>`
- `SoulWeb: <pawn> has spent its 12 triggered effects; <op> of <row> not run (said once per press)` (a Warning: only ops that run inside another op's handler draw on the budget)
- `SoulWeb: op '<op>' (first seen on <row>) has no handler; it does nothing (said once)`
- `SoulWeb: <pawn> cheats death (<row>): left at 1 health, ready again in <n> s`, then `SoulWeb: <pawn> rises: +<n> health`
- `SoulWeb: <pawn> is untargetable for <n> s`
- `UKitTables: <n> node(s), <n> abilities, <n> state(s), <n> ailment(s), <n> placed, <n> cue(s) over <n> soul(s)` at the first read; `UKitTables: no Data Table at '<path>' ...; read as empty` when one is missing.

## Tests

65 headless automation tests. `SandboxARPG.Kits.Rules.*` (seven: the two pools, the active refusals and gates, rings, neighbours and keystones, give-back, all-or-none lists, rank values, overrides), `Kits.Tables`, `Kits.Web.*` (ranks change moves, attributes and hits; ops run on their events), `Kits.Ops.*` (conditions; a press fires twelve triggered effects), `Kits.OpsExtra.*`, `Kits.OpsLate`, `Kits.Caps.*`, `Kits.States`, `Kits.Ailments`, `Kits.Phases.*`, `Kits.Placed.*`, `Kits.Minions.*`, `Kits.Mechanics.*`, `Kits.Mech2.*`, `Kits.Mech3.*`, `Monsters.MovePicker.*`, `UI.SoulWeb.*`, and `Kits.Vocabulary.BuiltNamesHaveHandlers`, which checks that every name `Data/Authoring/Souls/vocabulary.json` calls built has a handler. Sources under `Source/SandboxARPG/Kits/Tests/`, `Monsters/Tests/MonsterMovePickerTest.cpp` and `UI/Tests/SoulWebScreenTest.cpp`.

## What is not built

Every node can be taken and shows its text. How much of the text the game enacts is the row's `Built` column, shown on the screen: **full** (everything the text says happens at every rank), **partial** (the row's note says what is missing), **none** (shown as "Not built yet: ..."; nothing acts). `python scripts/check_soul_webs.py` prints the count. As of this page:

| Soul | Nodes | Full | Partial | None | Abilities full | Abilities partial |
|---|---|---|---|---|---|---|
| Creepy | 90 | 31 | 43 | 16 | 3 | 4 |
| Corpse | 87 | 40 | 43 | 4 | 5 | 2 |
| Ogre | 92 | 27 | 41 | 24 | 3 | 4 |
| Oni | 91 | 47 | 32 | 12 | 2 | 5 |
| Gargoyle | 89 | 26 | 54 | 9 | 2 | 5 |
| Golem | 90 | 42 | 38 | 10 | 5 | 2 |
| Eloko | 91 | 30 | 41 | 20 | 3 | 4 |
| Treant | 87 | 21 | 41 | 25 | 2 | 5 |
| Vampire | 90 | 39 | 39 | 12 | 3 | 4 |
| Vetala | 91 | 46 | 29 | 16 | 3 | 4 |
| **Total** | **898** | **349** | **401** | **148** | **31** | **39** |

The table is `python scripts/check_soul_webs.py` after the engine fault passes of 2026-10-04 (the leap, touch and signed-fraction fixes, then the op dispatcher fixes: chain hops that plant, per-pawn conditions, `before`, the `Pull` op toward a point, `ScaleByStat` into an attribute). The marks under it are as of the proof pass of 2026-10-04, which drove every ability and every mapped node through headless games and corrected the marks to what the logs showed (the Treant's corrections were applied the same day on Jon's word; its twelve proposed data fixes are not applied). The node counts leave out the six Unlock rows and the centre row each soul also has. No ability is unmapped. Six full or partial nodes use a word the vocabulary marks `built: false` (Corpse 1, Ogre 1, Eloko 3, Treant 1); the three such words are the op `SwallowPlaced`, the condition `HitAboveFractionOfTargetMaxHealth` and the attribute `MinionDamageIncreased`, each with a handler or a read in code that no proof run has exercised.

Also not built, from notes in the code:

- Allocations are not saved (by rule, above).
- A request the server refuses has no channel back to the client beyond the unchanged replicated state.
- A pull along a dash, onto a line, or toward the first enemy a chain touched (the `Pull` op draws toward the user, the aim point or a placed thing; `Kits/KitOpsBuiltin.cpp`).
- `ExtendBuff` and `ExtendState` cap the time left, not the time added; an `OnStateGained` op that adds the state it answers runs again down to the chain depth bound.
- Seconds consumed as an op amount, and per-stack scaling of reach on a move row (schema 7.8).
- Nothing here has run in a packaged build, in a joiner's window watched by a person, or on a dedicated server.

## The incoming hit, in order (2026-10-04)

A hit a pawn takes is shaped in this order after mitigation: the pawn's `DamageTaken` rules, the redirect to minions or placed things, the absorbing meters (a health pool), then `CheatDeath` on what is left for Health. A cheat-death's own conditions are read before the meters, as the hit found the pawn, so "while you hold any Vitae" is asked of the pool before it is emptied. A kill tells `OnKill` with the killing hit attached when it was the killer's own, so a condition about the hit or a `ShareOfHit` reads it. A standing `BonusDamage` rule may give `IncreasedIfCrit`; it cannot ask `HitIsCrit`.

## For engineers

`Source/SandboxARPG/Kits/`: `KitTypes.h` (the row structs of the six tables), `KitTables.*` (`UKitTables`: config paths, one load, one index per soul, `SetTestRows` for tests), `KitRules.*` (`FKitRules`: the allocation rules as pure functions), `SoulWebComponent.*` (`USoulWebComponent` on the pawn: ranks per soul replicated push-model, `RequestAllocate` and `RequestRespec` with their validated server RPCs under the `Tree` rate category, a list of 1 to 64 entries of 1 to 32 ranks each, `ResolveMove`, `Dispatch`, the standing-rule reads `GatherOutgoingBonus` and `FilterIncomingHit`), `KitOps.*` (`FKitOps` and the registrars), `KitOpsBuiltin.cpp`, `KitOpsExtra.cpp`, `KitOpsLate.*` (handlers; the caps as `FKitCaps`), `KitRowOverrides.*`, `KitAbility.*` and `KitAbilities.*` (`UKitAbility` and one name-only subclass per ability, generated), `KitSeams.h` (the function-pointer seams between the cue, phase, placed and minion code), `KitCues.*`. The legacy tree's lookups route to the web for a v2 soul (`Progression/SoulTreeComponent.cpp`). The screen is `UI/SoulWebScreen.*` and `UI/SoulWebLayout.*`, C++ UMG with no asset, opened by `AMonsterHUD` in place of `USkillTreeScreen`; its numbers are `Web*` rows of `UHudLayoutSettings`. No GAS type appears in `Kits/`: attributes, states and hits go through the wrapper ASC (CLAUDE.md 6.7). Provisional markers: register row `soul webs`.

## The ops pass of 2026-10-04

What the per-soul data passes reported and the ops, events and conditions cluster changed; the data contract is `docs/design/soul-kit-schema-v2.md` section 7.12.

- **The swing's count** (`TargetsHitAtLeast`, `PerTargetHit`) is of the move's own hits; a hit an op made in between, a placed thing's hit and a splash are not counted, and `OnSwingEnd`, a leap's `OnDashEnd` and a channel tick read the swing's own count.
- **A forced crit** with a `Count` covers every enemy of the swing it is spent on; a row with `bOneCritRoll` rolls once for a whole swing or landing; `BonusDamage` `NoCritRoll` takes the roll away.
- **An op's memory** (cooldown clock, armed threshold, per-target stamps, added seconds) is kept across a rebuild of the op lists, by node row and op index (`USoulWebComponent::OpMemory`).
- **A give-back** ends a lit toggle whose web is no longer unlocked.
- **Events**: `OnPlacedEvicted` beside `OnPlacedExpired`; `EventHasHit` for `OnHitOwnPlaced`; a Seed's pop is a placed hit; `HitEach` and a `Hit` with `OwnAbility` name an ability and are not that ability's own hit; `OnAilmentApplied` knows the ability of what applied the ailment (`HitAbilityIs`); a drag tells `OnControlApplied` (`Pull`); `OnShatter` carries the scaled shatter.
- **Conditions**: `SecondsSinceLastHitAbove`, `LastDashDistanceAbove`, `ChannelTickEvery` and `UseIndexEvery` read the event they are asked in; `RefundCooldown` and `ResetCooldown` take the names `Web` and `OtherThanWeb`.

Log lines: `SoulWeb: <pawn>'s cooldown of <row> is reset`, `Placed: <pawn>'s <row> #<id> gives way to #<id> (cap or budget); OnPlacedEvicted is told`, `Control: <pawn>'s stun lasts <n> s (was <n> s left)`, `KitPhase: <pawn> ends <toggle> (its web was given back)`.
