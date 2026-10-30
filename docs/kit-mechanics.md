# Kit mechanics

**Status:** first pass, provisional (register row `soul webs`). Ten pieces of bespoke C++ behind the [soul webs](soul-webs.md), one per signature mechanic of a soul, for what the general ops cannot express. Each quotes its soul's draft (`docs/design/souls/<soul>.md`) as its source; every number is a param of a node's row or a row of `DT_CombatRules`, all placeholders. The full word lists with every param are section 7.8 of `docs/design/soul-kit-schema-v2.md`.

## How a mechanic is reached

Nothing is keyed by a node id. A mechanic is plain C++ that data reaches by name, three ways:

- **Ops.** The mechanic registers ops like any other (`ConsumeIntoSeed`, `RouteLines`). A node's row names the op and its params.
- **Rule sets.** A node changes a mechanic's numbers with an op whose `op` is `Rule` and whose `trigger` is the mechanic's rule set (`PlagueRules`, `ArcRules` ...). A rule is never run and draws on no budget. When the mechanic acts it reads every held rule of its set whose conditions pass and **adds** their params, at the node's rank, to its own (`USoulWebComponent::ReadRules`). Which rules a read reaches is `FKitRules::HearsEvent` (schema 4.3): the rule sets a mechanic reads for one ability (`ArcRules`, `StokeRules`, `PlagueRules`, `SwallowRules`, `CurseRules`, `OverrideRules`) reach a rule on an ability web only for that ability's events, unless the rule says `"triggerAbility": "Any"` or names the other ability; the rule sets read with no ability (`LeechRules`, `ThresholdRules`, `AilmentRules`, `RemainsRules`) reach every held rule, whatever web it sits in, and `MinionRules` reaches a rule when the minion was raised by the rule's own ability or by none.
- **Events.** The mechanic tells the owner's web triggers of its own (`OnPlagueJump`, `OnCurseAnswer`), which ordinary ops answer.

Two kinds of hit a mechanic makes: a **share** (only the target's mitigation changes it; no roll, no crit, no `OnHit`) and a **final** number (consumed or reflected damage lands as it is). All server only. No tick: the mechanics that run over time share timers at four steps a second.

## Plague (Creepy)

A bite consumes the biter's poison on its target and seeds one strong instance of the ailment `Plague` (200 % of the bite's own base, plus what it consumed), which jumps between enemies by generations. Each host carries a real ailment instance, paid by the ailment clock like any other and counted as Poison (the row's `CountsAs`), so the digit, the poison lines and every node that reads poison see it. One manager per world (`UKitPlagueSubsystem`) keeps the lineage alone: its hosts, the generation, the value each instance had as made, and a beat a tick apart on which a jump is rolled to the nearest enemy of the biter in reach that the lineage never infected. A jump is a copy (no damage line again) at a share of the host's value as made, over a fresh full duration, one generation on. A host ends when it dies or its instance is gone, run out or consumed. A host carries the shared state `Plagued` (the bitten one `PlagueFirst` too); the biter carries `PlagueAlive` while any lineage of its own lives.

- **Ops:** `ConsumeIntoSeed` (on a hit: `HitShare` of the hit's base, `ConsumedShare` of what was consumed, `Flat`; with `Generations` a lineage, without it one plain instance; duration and tick from the ailment row `Plague`, 8 s and 1 s), `PlagueJump` (a jump from the nearest host in `Radius`, `Free` for no generation lost), `PlagueFirstHostKill` (a bitten host killed by a move's hit ticks every living descendant once).
- **Rule set:** `PlagueRules` (generations, what each keeps, jump chance and radius, host cap, jump on death, to every enemy in reach and at full value, never on a beat, incubation, first-host bonus, frothing, one lineage, a bite lock; at the bite: roots beside the bitten host, no consume, a larger seed, a placed store drawn into the seed; a cooldown and mana refund by the hosts a lineage reaches).
- **Event:** `OnPlagueJump`.
- **Caps rows:** `PlagueGenerationCap` 5, `PlagueHostCap` 24.
- **Leaves out:** a Plague tick is a share, so it fires no `OnHit` and plants nothing. An enemy is infected once per bite and never refreshed. Only the two marker states replicate.

## Static arc (Oni)

A critical strike on an enemy carrying a counted state spends the stacks and fires one arc per stack to a different enemy near the target, nearest first, each a share of the strike after the attacker's lines and the crit.

- **Ops:** `StaticArc` (on a crit; the state, the damage type, `Share`, `Radius`), `Bolts` (one triggered bolt per stack of a self state on a random enemy near the user; `Single` makes one bolt of all).
- **Rule set:** `ArcRules` (share, radius, how many stacks a strike spends, a spare arc returning to the target, a ground share, forking at the cap, one batch crit roll, an ailment each arc applies, a state gained per stack spent, another damage type).
- **Caps row:** `StaticArcHitCap` 10.
- **Leaves out:** an arc is no landed hit: it cannot crit by itself, plants no Static, grants no Tempo and releases no arcs. With fewer enemies than arcs the spare arcs are lost unless a rule returns them.

## Stoke (Oni)

A refresh with a snapshot for an aura the user keeps lit. The op reads the user's attack speed bonus once and writes two self states: `Stoked` (the window; its row's `Life` is how long a Stoke lasts) and the meter `StokeHeat` (the percent of increased damage the aura's ticks take), which the aura's own `BonusDamage` rule reads while `Stoked` holds. Nothing is read again until the next Stoke. A new Stoke spends the old one first.

- **Ops:** `Stoke` (the aura's row must be a lit toggle), `StokeToll` (the aura's self-damage a second time while Stoked), `StokeBurst` (so many of the aura's hits at once within its radius, and the Stoke is spent).
- **Rule set:** `StokeRules` (what a point of attack speed is worth, extra read from enemies inside, missing health or crit chance, a longer window per stack of a state, a weaker Stoke adding seconds and replacing nothing).
- **Caps row:** `StokeHeatCap` 1000.
- **Leaves out:** a Stoke on an aura that is not lit writes nothing.

## Swallow (Ogre)

A monster soft enough (holding the shred stacks, or under a health fraction) is taken out of play by `UKitSwallowSubsystem`: hidden, no collision, its movement held, its brain paused, untargetable, its ailments still ticking. It takes a share of the eater's maximum health each second and comes back at the eater's feet stunned, or where a spit lands it. It stays its spawner's monster and dies by the eater's hits, so the slot, the kill credit and the drops are as for any kill. The eater carries the self states `BellyFull`, `HandFull` and `StoneBelly`.

- **Ops:** `Swallow`, `DevourEnd`, `SpitBelly` (the belly's contents fly along the aim to the first enemy on the line), `SwallowPlaced` (a placed thing in the belly), `ThrowPick` and `ThrowLand` (a stunned monster rides a lob in the hand), `StealDefenses` (the eater wears a share of the target's Defense and resistances, one meal at a time).
- **Rule set:** `SwallowRules`.
- **Events:** `OnEat` (`Whole`, `Chunk`, `Stone`), `OnBellyEmptied`.
- **Caps row:** `SwallowHoldCap` 8 s.
- **Leaves out:** never a Boss, an Elite, a player-controlled pawn or a pawn the eater is not hostile to (the swallowed ally of the draft is cut); one thing at a time. A Boss or a player holding the stacks loses them instead (a Chunk). In a field monster's hands it is the bite only. The vocabulary marks `SwallowPlaced` as not built: its handler is registered and no proof run has exercised it.

## Faultline routing (Golem)

Geometry on top of placed lines ([placed things](kit-placed-things.md)). A Faultline deals no damage by itself: the Golem's ground spells run along it, pass to the lines that touch it, or erupt it.

- **Ops:** `RouteLines` (a ground hit meets the user's lines: a run line hits within reach and passes to lines within `Touch` of it while jumps are left; then the cast leaves its own line), `EruptLines` (every line erupts, oldest first, and is consumed), `DrawLineTo` (a line from the feet to a target, or from a placed thing to the enemy nearest its centre), `HitLinesTouching`, `PlaceAlong`, `HitEach`, `AddStateByHit` (a state grows by what a hit was worth before the holder's Defense and block).
- **Standing rules:** `LineBonus`, `LineCrossFull`, `StressCap`, `NoPushOut`, `NoOwnLines`.
- **Events:** `OnLinesErupted`, `OnPlacedPulse`; an eruption is told as `OnPlacedBurst`.
- **Caps row:** `AreaOverlapShare` 0.3 (an enemy on several lines takes the further ones at the share).
- **Readings of this pass:** two lines touch when their spines come within `Touch` (60 uu when the op names none); a Stress line is a Faultline wearing the tag `Stress`; a shock is a Triggered area hit, an eruption the ability's own Direct hit.
- **Leaves out:** a standing rule here is read by the mechanic with trigger `Always`; its conditions are not read.

## Seeds and runners (Treant)

What grows from placed Seeds. Every Seed op goes through one spend rule: an Old Seed refuses its first spend, a Seed Vault leaves it lying, a Volunteer grows back.

- **Ops:** `PopSeeds` (ripe Seeds burst; told as a burst), `Sprout` (the oldest ripe Seed becomes a minion where it lies, while the minion budget has room), `Runners` (each Seed has a runner to its nearest neighbours; an enemy crossing one is told), `SeedHops` (a vine hit near a ripe Seed hops through it to the nearest untouched enemy, with side shoots), `RipenSeeds`, `MarkGreatSeed`, `DropSeed` (its `MaxPerUse` counted for the cast its hit came from, so a late hit of an older cast spends that cast's Seeds and not the newest one's), `CastFromSeed`, and for Sap `AddStateToMinion` and `CollectFromMinions`.
- **Standing rules:** `SeedVault`, `OldSeed`, `Volunteer`, `Rootstock`.
- **Events:** `OnRunnerCrossed`, `OnSeedHop`, `OnMinionStateCollected`.
- **Readings of this pass:** a runner is the straight segment between two Seeds, and an enemy within `Width` of it has crossed it; a pop is a Triggered area hit, so the nodes that answer a burst answer a pop.
- **Leaves out:** Mycelium ("any ability you aim may be cast from a Seed") is built for the vine alone, as one hit from the Seed nearest the aim on an enemy the body could not reach.

## Stone Time (Gargoyle)

A stun's length is written on its target as a counted state, and the longer of an old and a new length is kept. The ops write those lengths and move them about.

- **Ops:** `WriteStateFromControl` (the target gains what a control has left on it as a state; on a Boss the length is written without the stun, with a slow), `MatchControlToState` (a control ends when the state it stands for is spent), `GatherState` (every count in reach is given up and the sum is paid, final, into the enemy in front of the user), `ShareLongestState` (a share of the longest count in a cone is written onto others that hold less), `HitPerStack`, `ControlPerStack`, `RefundPerStack` (per stack of a self state), `EndToggle` (a toggled stance ended from data, for example on a step).
- **Readings of this pass:** the length written is what the engine's own stun has left the moment it lands, so the applier's lines and the target's resistance are in it; a copy is a write, so the longer is kept.
- **Leaves out:** Pull the Keystone pays the gathered sum straight into the enemy in front of the user; it is not first held there.

## Riddle ledger (Vetala)

A curse the owner poses on an enemy, named by the state row that marks its bearer. It writes a share of what the owner's hits take from the bearer's Health on a Tally under a cap, counts each different kit ability that hits it as a Clue, runs a fuse, and gives its Answer when the fuse ends, when it is called, or when the bearer dies: the Tally times (1 + a share per Clue) on the bearer as a final hit, with a splash.

- **Ops:** `PoseCurse`, `CurseWrite`, `PauseCurses`, `CopyCurse`.
- **Rule set:** `CurseRules`.
- **Event:** `OnCurseAnswer`. **Conditions:** `CurseRanWhole`, `CurseAtFullClues`, `TargetHasCurse`.
- **Readings of this pass:** a source is an ability row; the damage written is what reached Health.
- **Leaves out:** Answer damage is never written on a Tally; a basic attack is no Clue unless a `CurseWrite` says so; a curse replaced by a newer one over the limit gives no Answer; a copy never copies.

## Remains (Vetala)

What an enemy leaves when it dies: a mark on the ground that abilities target, consume, stand on or pocket. Never an actor per corpse: each Remains is one placed thing of the row `Remains`, owned by the player it counts for, so the cap, the life, the replicated list and the ground marks are the placed system's own. The row is outside the per-player budget of placed things. A Remains remembers the dead enemy's maximum health.

- **Ops:** `SeatInRemains` (the user arrives at a Remains, heals, and where it stood its area hit lands), `ConsumeRemains`, `LeaveRemains`.
- **Rule set:** `RemainsRules`. **Event:** `OnRemainsConsumed`. **Conditions:** `SelfOnRemains`, `RemainsInReach`.
- **Caps row:** `RemainsRadius` 1500.
- **Leaves out:** a death leaves a Remains only for a player within the radius whose worn soul has an ability tagged `Remains`. The draft calls Remains a shared world object and asks for Jon's yes on that; until a soul tags an ability, other souls get none.

## Bewitch (Eloko)

One button. Pressed on a wild non-boss monster in range that holds enough of a state, it takes the monster over as a **Thrall**: the same field monster on the user's side, with its own row and moves, until killed; the state is spent, the monster healed, its kill paid once. Pressed anywhere else it commands: every Thrall fires its move with the most flat damage at the aim. A press that does neither is given back.

- **Ops:** `Bewitch`, `ArmTakeOver` (for some seconds the next take-over needs fewer stacks and has no cooldown), `EatThrall` (a Thrall dies; the user heals, gains stacks, and its next command lands the Thrall's strongest move once).
- **Budget:** Thralls are counted in points of their own (a Normal and an Elite cost different points) under a point budget and a cap on bodies; a take-over past either replaces the oldest.
- **Reading of this pass:** "strongest move" is the move with the most flat damage on its row.
- **Leaves out:** never a Boss, a player or another player's minion.

## Shared leftovers

Three small files serve several souls. `Kits/Mechanics/KitAilmentVerbs.cpp`: an ailment instance's age and how long it has been carried, `MoveAilment`, `TapAilment`, the rule set `AilmentRules` ([states and ailments](kit-states-and-ailments.md)). `Kits/Mechanics/MinionVerbs.cpp`: `MinionRules`, `TendMinions`, `SpendPlaced` ([minions](kit-minions.md)). `Kits/Mechanics/KitMech3.*`: shatters counted per press (the conditions `SpentThisPressAtLeast`, `ShattersThisPressAtLeast`, `OncePerPress`, `MaxPerUse`), the rule sets `LeechRules`, `ThresholdRules` and `OverrideRules`, and the trigger `OnMissTaken`.

## The words of Block E (E-2.262, 2026-10-27 to 2026-10-28)

The nine trees of CT-2.66 asked for about 175 new engine words, schema sections 7.83 to 7.263 of `docs/design/soul-kit-schema-v2.md`, each named after the node text that asked for it and each with a test. They are general: a word built for one soul's node is open to every soul. Grouped by what they act on, one line each:

- **Placed things** (`PlacedRules`): a thing's end as a place and a crumble that hurts; the first thing on an empty field; the third launched or broken in a window; a thing that grows as it sits; things dropped along a dash; more things a stack; lines between placed things; things laid apart, ahead, or as if they had sat; a thing that throws at random inside itself; a volley cast again from a placed thing (`VolleyRules`).
- **States and meters** (`StateRules`): a meter worth a share of maximum health; a stat scaled from a state; a fading meter that heals; a buff that lasts until you walk; what a buff's Defense prevented; stacks that keep their applier; how fast a state drains and what it reaches, by its applier; a state set to a number, kept past its end, spent by a late amount, or on one pawn alone; a pile that cannot fall off; a flat per stack.
- **States on stats that start at zero**: a state's per-stack Increased or More on such a stat (move speed, attack speed, area and the other fractions) now goes through the one rule a node's modifier already used (`FKitRules::AppendAttributeLine`), so Limescale slows and Sated speeds up where they did nothing.
- **Hits** (`HitRules`): a hit that cannot miss, be dodged or blocked; no crit at all; a hit typed by name; a hit along the whole line; a path hit again; a circle hit that leaves out its own target; a bonus by the wind-up or the control a hit asks for; a hit rule that reads the target's hit clock.
- **Poise and wind-ups**: a floor under Poise, a Poise that cannot miss, a hit in any wind-up told, the next wind-up out of a walk.
- **Ailments**: an explosion, its reach, one that hurts its user and a chain of them (`ExplosionRules`); a kept instance, every Nth instance, one made again as it runs out, a hit as it runs out; a tick worth more with time left; a cap for one ability's instances; a tick that names its ailment; an instance passed to a neighbour; a Plague host's death told.
- **Controls**: a control that stacks, that lasts longer, that holds a Boss, that is passed on by a dying pawn or matched to a count; a refused knockback sent back; a cleanse of your own controls; a target that ignores a control kind.
- **Heals and flasks** (`HealRules`, `FlaskRules`): every heal of one ability scaled; no health on hit; a restore you keep a share of; a flask that pours past full, a flask step read as it pours, a flask that is gone.
- **Consumes and refunds** (`ConsumeRules`): a consume of so much at most, counted from one enemy, paid as a crit or paying more under its rules; a refund of what is left or of the longest cooldown; a spend that leaves some, or only reads.
- **Movement and bodies** (`BodyRules`, `DashRules`): walking through enemies and being told of each; enemies chained to your walk; a holder a charge cannot pass; a dash that stops at the first enemy, that a Boss stops, or that ends beside a marked enemy; a swing you walk through; a turn rate attribute.
- **Chains, aftershocks and rings** (`ChainRules`, `AftershockRules`): a chain's end pull, a chain of shares of the first hit, an extra Static arc jump; aftershocks that come sooner and wider; a ring after other rings; the nearest few round a target.
- **Allies, minions and taunts** (`AllyGuardRules`, `TauntRules`): a guard that takes a share of an ally's hits; a taunt that holds a Boss; a minion near you, alone, or young; controls and ailments handed to a minion; a sacrifice of the nearest minion and a regrowth that knows why it died.
- **Attributes and gem lines** (`AttributeRules`): gem lines and bonuses of an attribute that do nothing, for a keystone's price.
- **Uses and timing**: an ability that uses itself and cannot be pressed; every Nth time, on one target or every third cast; an interval of its own; a run of uses; a use aimed near the last; the last swing; a recast chain that waits for a hit.

A channel's per-tick price now pays through the same mana-to-health conversion as the cast (`FKitLate::PayChannelTick`), so Blood Rider's Gyre spins.

## For testers

`Slime.Curse.Show` prints every live curse. The rest is reached through the general commands: `Slime.Kit.Op <Op> ...` runs one op as a rank-1 node would, `Slime.State.Show` shows `Plagued`, `Stoked`, `BellyFull` and the other marker states, `Slime.Placed.Show` shows Faultlines, Seeds and Remains, `Slime.Minion.Show` shows Thralls and Saplings.

## What is not built

The mechanics exist and their ops are registered, but a node only does what its row maps. The coverage of each soul's nodes (full, partial, none) is on the [Soul webs](soul-webs.md) page, and the gap is mostly there, not here. The `about` text of `Data/Authoring/Souls/vocabulary.json` says the mechanic classes of schema 7.6 are not built, while this folder holds them and their words are listed as built in the same file. None of the ten has been seen on screen by the team, and none has run with a joiner.

## For engineers

`Source/SandboxARPG/Kits/Mechanics/`: `KitMechanics.*` (`FKitRuleSum`, `FKitMechanics::ReadRules`, `LandShare`, `LandFinal`), `MechanicsCommon.h` (flat segment geometry, pure), `Plague.*`, `StaticArc.*`, `Stoke.*`, `Swallow.*`, `FaultlineRouting.cpp`, `SeedsAndRunners.cpp`, `StoneTime.cpp`, `RiddleLedger.*`, `Remains.*`, `Bewitch.*`, `KitMech3.*` (the one-line hooks the wrapper ASC, the web component and the field monster call, and the pure rules). No GAS type in the folder (CLAUDE.md 6.7); the wrapper stays four types (6.8). Tests: `SandboxARPG.Kits.Mechanics.*` (four: the pure rules, Static arc, Plague, Swallow), `Kits.Mech2.*` (three: lines, runners, Stone Time), `Kits.Mech3.*` (eight: the Tally, the Answer, the fuse hold, Thralls over the limit, the shatter chain, leech under a ceiling, Remains, the verbs registered). Provisional markers: register row `soul webs`.
