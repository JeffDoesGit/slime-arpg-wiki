# Level and growth

*Built in roadmap item E-2.8. Design rule: GDD 2.4 is still open; this follows the D-2.2 memo's recommendation and is marked provisional until the decision lands.*

## The idea

Your character has a level. Level 1 is the same for everyone: a fixed amount of health and mana. Each level above 1 adds a fixed amount from your character, plus a smaller amount from every soul you have slotted, primary or secondary. Souls do not change where you start; they change how fast you grow.

The formula, in words: maximum health is the base health plus, for every level above 1, the character's health growth and the health growth of each slotted soul. Maximum mana works the same way with the mana numbers.

Two things follow from the formula being recomputed rather than banked:

- **Swapping a soul is free of history.** Slot a different soul at level 20 and your maximum recomputes as if that soul had been there all along. Nothing depends on the order you wore souls in.
- **Swapping keeps your fraction.** At half health, swapping to a soul with more growth leaves you at half of the new maximum, not at a fixed number.

A level change fills health and mana to the new maximum. Whether a level-up should heal is still open; filling is the placeholder.

## Experience

Killing a field monster earns experience for whoever landed the killing blow. Each monster type has a fixed reward in its data row, 20 for now. Experience counts toward the next level only: you need 100 times your current level to move up, and the counter starts again from zero when you do, so level 2 costs 100, level 3 costs 200, and so on. There is a level cap, 50 for now, past which nothing accrues. A level-up fills health and mana like any other level change. The kill goes to one character; there is no sharing between players yet, and a monster that kills a player earns nothing. None of this is saved between sessions.

## Dying

Death costs a tenth of the experience you had gathered toward the next level, rounded to the nearest point, and never a level: at 60 of 200 you lose 6 and stand at 54. With nothing gathered you lose nothing. Your screen dims with "You died", the points lost, and a three-second count, then you are back at the start of the level with full health and mana and everything you carried. The share is a placeholder from the design memo; where you come back is the level's start until the town exists.

Since E-2.46 the death screen ends in a choice rather than a timer: after the countdown, two buttons, **Checkpoint** and **Town**. Checkpoint puts you at the start of the zone you died in, the entrance you came in through, or the forest's start if you never left it; Town puts you at the town's gate from the forest. You stay dead until you choose. Either way you come back with full health and mana.

## The numbers right now

Placeholders from the memo, to be tuned in play. They live in a data table and on the soul assets, not in code.

| Value | Placeholder |
|---|---|
| Character base health | 100 |
| Character health growth per level | 10 |
| Character base mana | 50 |
| Character mana growth per level | 5 |
| Creepy soul health growth per level | 5 |
| Creepy soul mana growth per level | 2 |
| Experience per field monster kill | 20 |
| Experience to leave level L | 100 × L |
| Level cap | 50 |
| Experience lost on death | 10% of progress toward the next level |
| Corpse soul growth | 0 (not set yet) |

Worked example at level 10 with the Creepy as primary and nothing in the secondary slot: health 100 + (10 + 5) × 9 = 235, mana 50 + (5 + 2) × 9 = 113.

## What you see

You spawn with full health and mana at level 1, so the orbs on the [HUD](hud.md) are full from the start. Above the ability bar a thin gold bar fills with your experience and a number beside it shows your level; both move only after the server has confirmed the kill. Equipping or removing a soul at level 1 changes nothing. At higher levels the maxima move with your souls.

Field monsters are not on this system. Their health, mana and defense are rows on the monster's own data ([Field monsters](field-monsters.md)).

## The skill tree

Each soul carries one tree with two halves. The **skill region** is live while the soul is your primary and holds its four moves and the nodes that change them; the **passive region** is live while the soul is your secondary and holds defence, sustain and movement. Since the passive-trees pass of 2026-09-26 (E-2.102 to E-2.109, CT-2.20) both trees are complete on paper: 26 Creepy nodes and 27 Corpse nodes, each with its rule built unless it waits on the ailment family (Reeking Shroud, Sure-footed, Heavy Bones, Molt's cleanse).

**Points.** You get one skill point per level from level 2, one pool, spent on either tree (Jon's rule of 2026-09-26; the D-8.2 memo said one of each kind and Duilio decides). Level 1 has none: a fresh soul fires only its **default attack**, the free first node, drawn with one generic icon on the bar and the tree. **Your other moves unlock by taking their nodes** (`Slime.MovesUnlockByNode`, on by default; 0 gives GDD 2.3 as written, the whole moveset on equip). The picker lists only what is unlocked; a locked move pressed by console is refused naming its node.

**What nodes do.** A node with numbers changes your attributes while its soul sits in the slot (Bone, Chitin, Thick Hide, Marrow Reserve, Slow Blood, Shamble, which now moves your walk speed). A node with overrides changes a move's numbers or shape while allocated (Long Claws, Frenzy, Quick Spit, Corrosive Spit, Wide Fork, Festering Rend, Clinging Shroud, Skittering; Restless Marrow, Deep Splinter, Both Hands, Bone Choir's decay). A node with a verb acts on an event: a landed claw (Marrow Claws, Tear, Buried Splinters), a shatter (Shrapnel, Brittle Bone, Bone Choir), a hit on you (Caustic Blood, Ossuary), a kill (Marrow Tithe, Grave Dust, Rot Feeder, Carrion), a health threshold (Rigor, Molt), standing still (Still Hunter's mana half), an enemy near you (Festering Shroud's contact stack, Miasma), Bone Ward starting (Warding Bones), the hands landing (Rising Dead leaves them in the ground), or the hit that would kill you (Unfinished; Hive Heart defers half of every hit and shrugs poison). Skitter and Lurch are moves the passive region grants outside the primary's kit; they show in the picker while their node is taken and have no animation yet.

**Poison** is the Creepy's ailment (E-2.109, on the D-3.5 memo): every application is its own stack with its own clock, ticking through the same damage path as a hit with no roll and no crit; one attacker holds at most five on one target, a sixth replacing the oldest; Rend copies the target's stacks with their remaining time, Festering Rend refreshes them instead, Carrion spreads a dying carrier's, Virulence and Slow Venom change your cap, duration and tick. The numbers are `DT_CombatRules` rows and the move rows.

The tree screen (P) shows one region per tab, node icons on the move nodes, the point pool, and Respec for testing. Everything here is provisional on D-8.2 and D-3.5; the exclusive pairs of the tree page are not built, so both sides of a pair can be taken.

## Regeneration

Since E-2.45 health and mana come back on their own. Once a second the server adds each character's regeneration rate to the current value, up to the maximum, and never while the character is dead. The base rate is two rows of the character rules table, `BaseHealthRegen` and `BaseManaRegen`; a gem line such as Mana regeneration or a self-buff such as Bone Ward adds on top. The design memo says characters have no base regeneration at all, so both rows are a playtest placeholder and the memo's answer is 0 in both.

## For testers

Kills raise the level on their own; two console commands on the host shortcut it:

- `Slime.SetLevel 10` sets the local player's level, clears experience and fills health and mana. Add a pawn name at the end to set another player's.
- `Slime.AddXP 250` awards experience with any level-ups it pays for, and logs the result. Same optional pawn name.
- Every kill logs `XP: <pawn> +20 for <monster> (L1 20/100)`; every death logs `<pawn>: died; lost 6 XP (L2 54/200); respawn choice open in 3.0 s` and the choice `respawn: <pawn> at Zone1/FromTown (checkpoint)`.
- `Slime.ShowAttributes` shows the resulting maxima on every monster.
- Skill tree: `Slime.SetLevel 4` gives three points of the one pool; `P` opens the screen; allocate Thick Hide with the Creepy secondary and `Slime.ShowAttributes` shows Defense 40. The host log prints `SkillTree: <pawn> allocated <soul>/<node> (points left N)` or `refused ...: <reason>`. Without the screen: `Slime.AllocateNode DA_Soul_Creepy ThickHide` allocates through the same rule the click reaches (a pawn name third for another player), `Slime.ShowTree` lists every node of every held soul as allocated, available or locked with the reason, `Slime.EquipSoul DA_Soul_Corpse secondary` fills the secondary slot, and `Slime.UseMove CreepySkitter` fires a node-granted move. All authority only, compiled out of Shipping.
- Regeneration: `Slime.SetAttribute Mana 0` and watch the orb climb `BaseManaRegen` per second; equip a gem with a Mana regeneration line and it climbs faster.

The table `DT_CharacterRules` must exist under `Content/Data`; if it is missing, the log warns and every base value falls back to zero.

## What is waiting on design

- The decision itself (D-2.2): whether base stats come from the character, the soul, or both.
- The experience numbers: reward per kill, the curve, whether a cap exists, and whether experience is shared in a group.
- Ability points per level and what they buy (D-8.2).
- Whether a level-up heals.
- How soul upgrades (D-4.1) interact with growth without counting it twice.

## For engineers

`Source/SandboxARPG/Progression/ProgressionComponent.*` on the player pawn: a replicated `Level` and `RecomputeVitals`, run on the authority at possession, on every soul slot change and on every level change; it derives only for player-controlled pawns. `Progression/CharacterRules.*` reads `DT_CharacterRules` by row name through a config path in `DefaultGame.ini`, the same pattern as the combat rules table. Growth rows are `HealthGrowth` and `ManaGrowth` on `USoulDefinition`. The write goes through `UMonsterAbilitySystemComponent::SetVitalMaxima`, so the GAS surface stays inside the wrapper. The tree is `Progression/SoulTree.*` (the node row, `FSoulTreeNode`, and read helpers), `Progression/SoulTreeComponent.*` on the pawn (allocations per soul, replicated; points derived from the level; `RequestAllocate` and the validated server RPC; effects through `ApplyModifierSet` while the soul is slotted), `UI/SkillTreeScreen.*` (the screen, C++ UMG) and `AMonsterHUD::ToggleSkillTree`. A soul names its table in `USoulDefinition::Tree`. Regeneration is `InitRegen` (base rates from the two table rows) and a one-second timer on the same component, `TickRegen`, which reads the current `HealthRegen` and `ManaRegen` attributes and writes the vitals' base values like every other vital write. `Progression/ProgressionConsoleCommands.cpp` holds `Slime.SetLevel`, compiled out of Shipping.
