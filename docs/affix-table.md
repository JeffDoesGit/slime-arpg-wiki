# Affix table

*Built in E-3.1 (branch `feat/affix-table`). The first piece of the accessory system.*

## What you see

Nothing yet. There are no accessories to pick up and no screen that shows a stat line. What exists is the list of every stat line an accessory will ever be able to roll, as a table the game loads.

## How it works, in plain words

The design review of 2026-09-07 approved 107 stat lines for accessories, in three groups: **offense** (58), **control and utility** (24) and **defense** (25). Flat fire damage, chance to poison, cooldown reduction, life leech, and so on. This item turns that list into a data table with one row per line.

Each row says which group the line belongs to, what it is called, which design rule it came from, which of your stats it would change, and how: **flat** adds a number, **increased** adds to a shared percentage, **more** multiplies on its own. Whether a line is flat, increased or more was assigned by a simple reading rule from the line's wording, so a designer can override any of them.

Two columns are deliberately empty or placeholders. The **damage type** column waits on the damage-types decision (D-3.1). The **target stat** column names real stats where they exist today (health, mana, defense, speed and their regeneration) and a placeholder name everywhere else, because most of the stats the pool refers to (critical chance, resistances, block) do not exist on a character yet.

The table has no numbers. How much a line rolls, how many tiers it has, which slots it can appear on: all of that is a separate decision (D-3.3) and lands as more columns later.

## Settled

- The list of lines and their three groups.
- One table, one row per line, editable by designers without code.

## Waiting on design

- Value ranges, tiers, prefix/suffix, slot restrictions, local versus global (D-3.3).
- Damage types (D-3.1).
- Six leftover questions from the review, each with a memo recommendation now (D-3.4): the name of the "mark" debuff, which enemy states get a damage line, whether minion lines stay, whether block exists without shields, what freeze magnitude means, and how lines reach a soul's moves.
- Accessories themselves, equipping them, rolling them and dropping them (E-3.2 to E-3.5).

## For engineers

`Source/SandboxARPG/Affixes/AffixRow.h` (`FAffixRow`, `EAffixCategory`, `EAffixOperation`). `Data/DT_Affixes.csv` is the import source, generated and checked by script (107 unique rows, group counts, every rule reference present in `docs/affix-pool.md`); `Data/README.md` explains the columns and the import. The asset is `Content/Data/DT_Affixes`, imported by a human in the editor. Tags are plain names, not gameplay tags, which stay inside the ability wrapper.
