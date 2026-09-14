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

Since E-2.47 every soul can carry a tree, and the character spends points in it. Press `S` (the key is in the HUD settings) to open the tree screen: your soul on the left with the points you hold, the tree drawn over the soul's orb in the middle, the node you clicked on the right with an Allocate button. Two tabs: **Skill region** shows your primary soul's tree, **Passive region** your secondary soul's.

Every character level from the second grants one primary point and one secondary point. Primary points buy nodes in the primary soul's skill region, secondary points buy nodes in the secondary soul's passive region; a soul has to be in that slot for you to spend on it. The first node of each region comes free with the soul. A node needs its earlier nodes first, and the keystone needs any two of three. Nodes stay on the soul: swap the soul out and back and they are still there; unspent points stay with the character.

What a node does today: a node with numbers on it (Chitin's armour and poison resistance, Thick Hide's armour) changes your stats the moment it is taken and for as long as that soul sits in its slot. A node that describes a behaviour (Molt, Skitter, the keystone, the four moves of the skill region) is allocated and shown, and does nothing yet; each of those is its own later piece of work. The moves of a soul are still all yours the moment you equip it, tree or no tree.

The Creepy has the first tree: four skill nodes and ten passive nodes, the ones in the design memo. No other soul has one yet; its tab says so.

Not built: soul rank as a gate on tiers (there is no rank yet), respec, the aura budget, and whether points should be held back while a slot is empty. All of it is provisional on the progression decision (D-8.2).

## Regeneration

Since E-2.45 health and mana come back on their own. Once a second the server adds each character's regeneration rate to the current value, up to the maximum, and never while the character is dead. The base rate is two rows of the character rules table, `BaseHealthRegen` and `BaseManaRegen`; a gem line such as Mana regeneration or a self-buff such as Bone Ward adds on top. The design memo says characters have no base regeneration at all, so both rows are a playtest placeholder and the memo's answer is 0 in both.

## For testers

Kills raise the level on their own; two console commands on the host shortcut it:

- `Slime.SetLevel 10` sets the local player's level, clears experience and fills health and mana. Add a pawn name at the end to set another player's.
- `Slime.AddXP 250` awards experience with any level-ups it pays for, and logs the result. Same optional pawn name.
- Every kill logs `XP: <pawn> +20 for <monster> (L1 20/100)`; every death logs `<pawn>: died; lost 6 XP (L2 54/200); respawn choice open in 3.0 s` and the choice `respawn: <pawn> at Zone1/FromTown (checkpoint)`.
- `Slime.ShowAttributes` shows the resulting maxima on every monster.
- Skill tree: `Slime.SetLevel 3` gives two points of each kind; `S` opens the screen; allocate Thick Hide with the Creepy secondary and `Slime.ShowAttributes` shows Defense 40. The host log prints `SkillTree: <pawn> allocated <soul>/<node> (primary points N, secondary points M)` or `refused ...: <reason>`.
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
