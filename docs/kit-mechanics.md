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

- **Ops:** `ConsumeIntoSeed` (on a hit: `HitShare` of the hit's base, `ConsumedShare` of what was consumed, `Flat`; with `Generations` a lineage, without it one plain instance; duration and tick from the ailment row `Plague`, 8 s and 1 s), `PlagueJump` (a jump from the nearest host in `Radius`, `Free` for no generation lost), `PlagueJumpFrom` (a hit on a host sends that host's own lineage onward at once, to the nearest uninfected enemy in the lineage's reach, with no roll; a host infected by the same hit is not jumped; the Creepy's Stirred Nest, at most once every 2 seconds a host; logged `Plague: lineage N jumps from X to Y (PlagueJumpFrom)`), `PlagueFirstHostKill` (a bitten host killed by a move's hit ticks every living descendant once).
- **Rule set:** `PlagueRules` (generations, what each keeps, jump chance and radius, host cap, jump on death, to every enemy in reach and at full value, never on a beat, incubation, first-host bonus, frothing, one lineage, a bite lock; at the bite: roots beside the bitten host, no consume, a larger seed, a placed store drawn into the seed; a cooldown and mana refund by the hosts a lineage reaches).
- **Events:** `OnPlagueJump`, and `OnPlagueLineageEnded` (since 2026-11-02, schema 7.265): when a lineage is over (its last live host gone and no wait on the biter), the biter's web is told once, with the Plague as the bite made it as the amount and every host the lineage ever had as the event's pawns; the target `LineageHosts` names those that still live. The Creepy's Last Swarm answers it: each living host takes one Poison worth half the Plague. A lineage whose biter is gone is told to no one.
- **Drawn stores** (since 2026-11-04, schema 7.271): the `PlagueRules` key `DrawStoreMore` multiplies what a bite draws from a placed store into the seed by 1 plus it, and only that part; the Creepy's Rot Garden doubles it.
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

## The words of the unwritten nodes (2026-11-02 to 2026-11-04)

Jon's review of the proposed texts for the trees' unwritten nodes gave the Creepy (pass 1, #596, and its partials in pass 2) and the Corpse (pass 2, #599, 25 nodes from his marks) their real nodes. The engine words they asked for are schema sections 7.265 to 7.275, each with a test; the Plague's two are in its section above. The ailment ones are on [states and ailments](kit-states-and-ailments.md).

- **A tick that asks for a bonus** (7.266). A standing `BonusDamage` rule now reaches an ailment's tick when its conditions hold `HitIsTick` or `HitIsForcedTick`, and only then; every other rule still leaves a tick as it was, and a tick still rolls no crit. With it `IncreasedPerAilingNearTarget`: so much increased for each other enemy within the rule's `Radius` of the target that carries your own instance of the named ailment (a Plague counts as Poison). The Creepy's Clustered Rot: 6 % a Poisoned enemy within 3 m. Rules on other souls that already asked for a tick (the Vampire's Vintage, the Vetala's Red Mist) now act.
- **Near an explosion's centre** (7.267). An explosion's hit carries where it went off; the condition `TargetWithinExplosionCentre` holds when the enemy's capsule edge is within that distance of it. The Creepy's Tamped: 10 % increased within 1 m. A kill carries no centre, so an `OnKill` op asking it never holds.
- **A placed thing's burst** (7.269). A hit knows when it is a placed thing's burst (`FHitSpec::PlacedBurst`), and the condition `HitIsPlacedBurst`, optionally naming the row, reads it. `HitIsPlaced` still reads pulses and enter hits. The Creepy's Brimstone raises Blight Bloom's burst by 10 %; before, it raised the pool's pulses and never the burst.
- **Every enemy a swing landed on** (7.273). The target `SwingTargets` names each living enemy the running swing's own hits landed on, once each; at `OnSwingEnd` that is the whole swing. An op's hit, a placed thing's and a splash add nobody. The Corpse's Whipcrack: a Spinal Tap that lands on 3 or more Cracks each of them twice.
- **Projectile speed** (7.274). The attribute `ProjectileSpeed` speeds every projectile whose hit is yours by 1 + it, never under a tenth of its launch speed (`AMoveProjectile::Init`): a kit row's, a shatter's shards, a placed thing's shots and the legacy spit alike. The range stays, so the flight is shorter in time; a lob keeps its flight time. The Corpse's Fletching gives +15 %. Log: `Projectile: <name> flies at S uu/s (B with <pawn>'s projectile speed +N%)`.
- **Action speed** (7.275). The attribute `ActionSpeed` and a column `ActionSpeed` on the move row each multiply a kit move's play rate after attack or cast speed, as separate mores: row rate x (1 + attack or cast speed) x (1 + the attribute) x (1 + the column), never under 0.1 (`FKitRules::RateWithSpeeds`). A stance or a command, which reads neither kind's speed, still takes it. The Corpse's Quick Claws adds 0.08 to Grave Clutch's column, so its clip, both claws and the commit come 8 % sooner.

The Corpse's Rattlebone Draught adds a state of its own to `DT_KitStates`, `Draught` (a self count, cap 1, 4 s, `D` on your own row): for 4 seconds after a flask is drunk, every Shatter of yours restores 4 % of your maximum mana.

## The words of the partial nodes, passes 3 to 5 (2026-11-04 to 2026-11-06)

Three passes over the nodes that did part of what their text says (#605) asked for schema sections 7.276 to 7.341, each with a test seen failing without its line. Like every word, each is open to every soul. The ones about states and ailments are on [states and ailments](kit-states-and-ailments.md), placed things on [placed things](kit-placed-things.md), minions on [minions](kit-minions.md), move columns on [move phases](kit-move-phases.md), the melee area line on [stats](attribute-set.md), and how a node's numbers are resolved on [soul webs](soul-webs.md). The rest, grouped by what they act on:

**Hits**
- **A hit worth the target's stacks** (7.276). `HitPerStack` with `OfTarget` counts the named state on the struck enemy, not on the user (`Kits/Mechanics/StoneTime.cpp`). The Eloko's Rude Awakening: 8 Lightning per Toll the enemy holds when its Entrance ends, its Tolls not spent.
- **A hit that rolls on from the aim** (7.278). `Hit` with `FromAim` 1 starts its shape at the aim and runs along the user's aim. The Eloko's Wave of Sound: a wave 10 m by 5 m on from the landing for 60 % of it.
- **Death rings by generation and count** (7.282). A `Hit` op at `OnEnemyDied` reads the dead enemy's last hit, when it was yours, as its parent, so death rings chain (`FHitSpec::ExplosionChain`); `MaxLinks` bounds the generations and `MaxRings` the rings of one chain, the first counted (`FKitElokoRules::EvaluateRingAllowed`). The condition `TargetRangOut` holds on an enemy one of your rings struck. The Eloko's Requiem: 3 generations, 12 rings. A death by another player's explosion starts a new chain.
- **Every Aftershock says so** (7.292). `FHitSpec::bAftershock` is set on a `Repeat` with `Aftershock`, a `Hit` op with `Aftershock` and an eruption's aftershock share; the condition `HitIsAftershock` reads it, and the repeat-share gate lets a rule that names it through on an Aftershock repeat. The Golem's Rubble Trouble, Echo Chamber and Sand Fall.
- **Thorns before Defense** (7.296). `Thorns` with `ShareOfRaw` returns a share of the hit as it was before the user's Defense, resistances and damage-taken rules (`FHitOutcome::RawDamage`), beside `ShareOfHit`. The Golem's Mirror Stone: 40 %.
- **A hit of a spell** (7.297). The condition `HitIsSpell` holds when the hit's kit ability row is of the kind Spell. The Golem's Bedrock, which no longer reaches attacks.
- **The enemy a landing hit hardest** (7.306). A swing keeps the pawn its own hits struck hardest (`USoulWebComponent::GetSwingHardest`); the target `SwingHardestHit` names it, and `Hit` `ShareOfSwingHardest` is a share of that hit before mitigation. The Gargoyle's Strike Twice: 50 % of the landing as Lightning, each landing of a recast chain its own.
- **What a swing took from an enemy** (7.308). The swing also keeps what its own hits took from each pawn (`GetSwingTaken`); `Hit` `FractionOfSwingTaken` is a share of it. The Gargoyle's Fissure: 60 % of what the blow's target took, to every enemy on the line.
- **A first hit's More its hops do not share** (7.305). `BonusDamage` `FirstHitOnly` puts its More on the first hit alone; a chain hop, a share of the first hit, is counted from that hit before the More (`FHitOutcome::FirstHitMore`, `RawForShare` at the share callers). The Treant's Briar Crown: 40 % less on the vine's first hit, none on its hops.
- **A hit that changes its type on some targets** (7.333). The standing op `HitTypeAs`: a rule of the attacker's web that holds for a hit of its ability on that target turns the hit into the named type at the top of `ApplyHit`, before anything reads the type, so that type's lines, the target's resistance (Physical: Defense) and penetration apply (`FKitLate::HitTypeAs`). Never a final number, an ailment tick, thorns, or a share that is not a repeat. The Gargoyle's Hard Water: a frozen or stunned enemy takes the water as Physical. Each tick reads the target as it stands then, so a tick after the stun ends is Cold again. Every hit of a pawn with a web scans its active ops once for it; nothing changes without a `HitTypeAs` op.
- **A flat of the user's missing health** (7.314). `BonusDamage` `FlatShareOfMissingHealth`. The Vetala's Deadfall: 16 % of missing health on the drop.

**Conditions and clocks**
- **A per-target window that ends on its last tick** (7.277). `OncePerTargetSeconds` takes a window as over 0.01 s before its full length (`FKitRules::WindowOver`, `WindowSlack`), so a 1 second window on a 0.25 s tick opens on every fourth tick, not every fifth. The Gargoyle's Limescale. The Eloko's own minion clock, `OncePerMinionSeconds`, is unchanged.
- **One clock for a node's ops** (7.283). The condition `NodeCooldownSeconds` is one clock for every op of the node (`USoulWebComponent::NodeReadyTime`), where `CooldownSeconds` is each op's own. The Eloko's Collection Plate, the Gargoyle's Counterweight.
- **While the user's channel runs** (7.311). The condition `SelfChanneling`, optionally naming the ability (`USoulComponent::GetKitChannelAbility`). The Vetala's Rip Cord.
- **A hit that takes you under** (7.290). `OnHealthBelow` carries the hit that drained the health (`UMonsterAbilitySystemComponent::GetDrainingHitOutcome`), so `EventHasHit` and `HitIsTick` can read it: a cost or a tick is not a hit. The Vampire's Turn to Bats.
- **OnControlApplied tells its duration** (7.288). The event's amount is the control instance's new length. The Vampire's Dusk: a Blind you apply also Slows the enemy for as long as it lasts (`ApplyControl` `DurationPerAmount`). No older `OnControlApplied` op read the amount.

**Uses, repeats, pulls and leech**
- **Leech reads its hit** (7.284). Every leech (a gem's, a row's `HealFractionOfDamage`, the `Leech` op) goes through `FKitMech3::Leech` with the landed hit, so `LeechRules` may ask the ability, the use and the hit's conditions; the condition `HitLeeched` holds on a hit whose leech after the rules was above 0, even when full health or the rate cap let none through. Every older `LeechRules` rule names no ability and hears every hit as before. The Vampire's Third Course, Dinner Guests and Seeing Red.
- **A restore per ailment instance, up to a ceiling a use** (7.286). The late amount `PerAilmentInstance` counts the user's own instances on the target; `Restore` `CapPerUse` stops it for the rest of that use (`FKitMech3::UnderOpUseCap`). The Vampire's Garnish: 1 mana a Bleed, 6 a swing.
- **A stat read at each ability's press** (7.320). `ScaleByStat` `ReadAtUse` on an op that names no ability now keeps one reading per ability, taken at its press; a hit reads its own ability's press, else the latest. The Golem's Bedrock, so an Epicentre's later pulses read the Defense of its press. The six older `ReadAtUse` ops all name their ability and are unchanged.
- **A wind-up counted in whole steps** (7.319). `BonusDamage` `StepSeconds` beside `IncreasedPerWindUpSecond` floors the wind-up to whole steps (`FKitPhaseRules::WindUpInSteps`). The Golem's Patience of Stone: 3 % a 0.5 s, so the Slam's 0.6 s gives 3 % and Where I Stand's 0.3 s none. The Ogre's Slow Hand names no step and is unchanged.
- **A repeat that follows the user** (7.312). `Repeat` `Fresh` aims each repeat from where the user stands then (`FKitRepeatRun::bFresh`). The Vetala's Top Heavy.
- **A repeat that waits for a state to end** (7.334). `Repeat` `UntilStateEnds` with a state as its arg watches the enemies the last landing hit that hold the user's state, and lands once when the first of them holds it no longer; `MaxDelay` is its guard (`USoulComponent::RepeatKitHitWhenStateEnds`). A spend to nothing ends a state as running out does; a part spend does not. The Gargoyle's Aftershock: 75 % of the landing when the first Stone Time ends. Every leap landing now remembers whom it hit.
- **A pull over seconds** (7.325). `Pull` `OverSeconds` splits its distance into quarter-second steps on a server timer, the first at once, each a pull of its own, on the pawns caught when the op ran (`FKitRules::PullSteps`). The Eloko's Supper Bell: 7 m over 2 s at a bell's first ring. Named `OverSeconds` because `OnInterval` reads `Seconds` as its period.
- **A pull at the walker's pace** (7.321). `Pull` `DistanceFromOwnSpeed` makes each step the user's ground speed times those seconds (`FKitRules::DistanceFromSpeed`); a still user pulls nothing. The Golem's Walking Wall carries the enemies in front of it as it walks.
- **A knockback aside of the hit's line** (7.328). The move column `bKnockbackAside`: the knockback goes across the hit's line (a projectile's flight) to the side the enemy stands on, not away from the user (`UMonsterAbilitySystemComponent::EvaluateKnockbackAside`). The Ogre's Bowling alone sets it.
- **A throw that carries a pawn flies as its body** (7.332). When the Ogre's throw holds a pawn, the volley's first projectile wears that pawn's main mesh in the clip it was playing, and no rock (`AMoveProjectile::InitCarried`, replicated). A body of several parts flies as its main mesh alone.
- **A meal that ends the swing** (7.281). `USoulComponent::InterruptMove` ends a running move for any reason (`InterruptForStun` is its stunned case); the Eloko's Eat the Help calls it after the meal.
- **An Answer called at the press** (7.317). The op `CallCurse` at `OnUse` calls a Riddled enemy's Answer at once and spends the use's volley, so no bolt is thrown. The Vetala's Answer Me.
- **An untargetable time held to a drift** (7.316). `Untargetable` `DriftRadius` sets the user back to the circle's edge when the time ends. The Vetala's Slip the Body: 4 m.
- **A client reads a health test** (7.285). The owning client now reads `SelfHealthBelow` and its named threshold (raised by held unconditioned `ThresholdRules`) in a node's `OverrideRules`, so the bar shows the cost and times the server will charge (`FKitRules::ClientHealthVerdict`). Display only; it reaches every soul's override rules under a health test. The Vampire's Tourniquet and Glutton.

- **A froth's swing, the look only** (7.309). A Plague host that froths now swings its own basic clip at the neighbour as a look (`UKitPlagueSubsystem::EvaluateFrothSwingMove`, `USoulComponent::PlaySwingLook`, which restores the host's re-fire and commit clocks on the server); the hit is still the biter's tick. Not while the host is busy or held. The Creepy's Frothing.

The Oni's Heat Haze and Crest needed no word: each is a `Cleanse` of the Oni's own controls on enemies (Blind inside the Mantle while it runs unstoked, Shock when Tempo ends); since #604 a cleanse that ends nothing writes no log line.

**What reaches other souls.** Most of these are read only by the rows that name them. These move older behaviour: a decoy with health is now worn by enemy projectiles, leap landings and lob bursts, so the Creepy's Shed Skin husk and the Corpse's Bone Totem too, and a lair Elite's totem by a player's ([placed things](kit-placed-things.md)); a statue's body blocks every walk and dash and stops the projectiles of every soul, the Gargoyle's own included ([placed things](kit-placed-things.md)); op overrides that multiply now land after the sets and adds ([soul webs](soul-webs.md)); the per-target window above opens a tick sooner for every `OncePerTargetSeconds` op; any `Place` with a target carries that target's size (read only by a blocking row); and the owning client's bar reads health tests.

## For testers

`Slime.Curse.Show` prints every live curse. The rest is reached through the general commands: `Slime.Kit.Op <Op> ...` runs one op as a rank-1 node would, `Slime.State.Show` shows `Plagued`, `Stoked`, `BellyFull` and the other marker states, `Slime.Placed.Show` shows Faultlines, Seeds and Remains, `Slime.Minion.Show` shows Thralls and Saplings.

## What is not built

The mechanics exist and their ops are registered, but a node only does what its row maps. The coverage of each soul's nodes (full, partial, none) is on the [Soul webs](soul-webs.md) page, and the gap is mostly there, not here. The `about` text of `Data/Authoring/Souls/vocabulary.json` says the mechanic classes of schema 7.6 are not built, while this folder holds them and their words are listed as built in the same file. None of the ten has been seen on screen by the team, and none has run with a joiner.

## For engineers

`Source/SandboxARPG/Kits/Mechanics/`: `KitMechanics.*` (`FKitRuleSum`, `FKitMechanics::ReadRules`, `LandShare`, `LandFinal`), `MechanicsCommon.h` (flat segment geometry, pure), `Plague.*`, `StaticArc.*`, `Stoke.*`, `Swallow.*`, `FaultlineRouting.cpp`, `SeedsAndRunners.cpp`, `StoneTime.cpp`, `RiddleLedger.*`, `Remains.*`, `Bewitch.*`, `KitMech3.*` (the one-line hooks the wrapper ASC, the web component and the field monster call, and the pure rules). No GAS type in the folder (CLAUDE.md 6.7); the wrapper stays four types (6.8). Tests: `SandboxARPG.Kits.Mechanics.*` (four: the pure rules, Static arc, Plague, Swallow), `Kits.Mech2.*` (three: lines, runners, Stone Time), `Kits.Mech3.*` (eight: the Tally, the Answer, the fuse hold, Thralls over the limit, the shatter chain, leech under a ceiling, Remains, the verbs registered). Provisional markers: register row `soul webs`.
