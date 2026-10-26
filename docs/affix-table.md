# Affix table

*Built in E-3.1, tiered in E-3.3. The list of stat lines a gem can roll.*

## What you see

Nothing yet. There are no accessories to pick up and no screen that shows a stat line. What exists is the list of every stat line an accessory will ever be able to roll, as a table the game loads.

## How it works, in plain words

The design review of 2026-09-07 approved 107 stat lines for accessories, in three groups: **offense** (58), **control and utility** (24) and **defense** (25). Flat fire damage, chance to poison, cooldown reduction, life leech, and so on. This item turns that list into a data table with one row per line.

Each row says which group the line belongs to, what it is called, which design rule it came from, which of your stats it would change, and how: **flat** adds a number, **increased** adds to a shared percentage, **more** multiplies on its own. Whether a line is flat, increased or more was assigned by a simple reading rule from the line's wording, so a designer can override any of them.

Two columns are deliberately empty or placeholders. The **damage type** column waits on the damage-types decision (D-3.1). The **target stat** column names real stats where they exist today (health, mana, defense, speed and their regeneration) and a placeholder name everywhere else, because most of the stats the pool refers to (critical chance, resistances, block) do not exist on a character yet.

Since E-3.3 the table has numbers, all placeholders. Every line exists ten times, once per tier, so the table is 1070 rows: `FlatHealth_T01` rolls 5 to 10 health, `FlatHealth_T05` 40 to 54, `FlatHealth_T10` 174 to 227. Each tier's band starts just above where the one below ended, so a higher tier is always a bigger number. A gem of tier IV can roll tiers 1 to 4 of a line and nothing above; see [gems](gems.md) for how a roll works.

Nobody types those 1070 rows. Two small sheets hold what a designer tunes: one row per line with its tier 1 band, how fast the band widens per tier, and the grid its values sit on (whole points, tenths, hundredths); and one row per tier with a knob that widens or narrows that tier for every line at once. A script turns the two sheets into the table and checks it. A line's display name is the whole line as the player reads it, in the word guide's form (`docs/design/word-guide.md` sections 3, 4 and 13; E-3.10), with `#` for the rolled number: `+# Physical damage`, `+#% Physical damage`, `#% more damage`, `#% chance to Burn`. A line on an attribute the game reads as a fraction rolls a fraction: critical chance, critical damage, attack and cast speed, accuracy, the penetrations, area of effect, ailment and stun duration, block chance and block value, healing received and the resistances roll 0.03 for 3 %. Until 2026-10-19 fifteen of them rolled whole points, so a critical chance line of 3 gave 300 %; gems rolled before then keep those numbers until they are rerolled or the saves are wiped. Eight more lines named an attribute that does not exist and did nothing; since 2026-10-19 they name the one the game reads, as fractions: dodge (capped at 75 %), leech (all leech together heals at most 20 % of maximum health a second, the `LeechMaxPerSecond` row, Path of Exile 1's rule; the rest is lost), cooldown recovery, skill duration, mana cost (capped at 90 %), damage taken, minion damage and minion health. Low-health damage reduction still does nothing. Since E-5.3 every line is also a **prefix** or a **suffix** (`Slot`), Path of Exile's convention as read in `Research Bank/2026-10-16-affix-prefix-suffix-conventions-research.md`: a prefix is what the gem gives on an empty character (a pool, a flat amount, a damage family, a magnitude), a suffix is how that power is shaped (a rate, a chance, a resistance, regeneration, on-hit, penetration, shred, a speed). Offense is 39 prefix families to 19 suffix, Defense 11 to 14, Control and utility 10 to 14; the script refuses a sheet where any colour has fewer than two families of a kind, since a cut gem must be two and two (D-5.4). The picks are provisional on the D-5.4 memo's Appendix A; which slots a gem can sit in and local versus global stay with D-3.3.

## Which lines do something (E-3.11)

The test `SandboxARPG.Affixes.EveryAffixProven` puts each of the 107 lines on a gem by itself, at the middle of its tier V band, equips it the way a player does, and measures the change the line names where the game reads it: a hit's number, misses or crits over a seeded run of hits, a vital, a timer, a move's cooldown or cost. Last run (2026-10-25): **58 working, 49 inert, 0 broken.** A working line moved its measure the way its name says; an inert line names a stat the game does not have yet, so it rolls and shows but changes nothing until its mechanic is built.

Four working lines meet less in play than the test shows:

- **The four penetrations** work, but every monster has 0 resistances today (`Data/DT_MonsterStats.csv`), so in play they change nothing.
- **Accuracy** works, but the base hit chance is 95% with no evasion on monsters, so it adds at most 5% of hits.
- **Stun duration** works, but a stun is cut by the hard-control clock, so it adds nothing to a stun already at that limit.
- **Increased and more Defense** work, but a player has no Defense of their own (only health and mana grow with level), so alone they multiply 0; they need flat Defense from another gem, a node or a buff.

| Line | Verdict | What the test measured |
|---|---|---|
| Accuracy | working | hits landed of 2000 seeded swings: 1899 without, 2000 with a line of 0.47 on Accuracy; the base hit chance is 95% with no evasion, so the line adds at most 5% of hits |
| AilmentDuration | working | a poison instance's seconds: 4 without, 4.76 with a line of 0.19 on AilmentDuration |
| AllResistances | inert | no attribute Resistance; its mechanic is not built |
| AreaOfEffect | working | Acid Spit's impact radius: 100 without, 147 with a line of 0.47 on AreaOfEffect |
| AttackSpeed | working | an attack's play rate: 1.5 without, 1.785 with a line of 0.19 on AttackSpeed |
| BleedDamage | inert | no attribute BleedDamage; its mechanic is not built |
| BlockChance | working | blocks in 2000 seeded hits taken: 0 without, 390 with a line of 0.19 on BlockChance |
| BlockValue | working | damage taken over 500 seeded hits with a 75% block chance: 1e+04 without, 6400 with a line of 0.47 on BlockValue |
| BurnDamage | inert | no attribute BurnDamage; its mechanic is not built |
| CastSpeed | working | a spell's play rate: 1 without, 1.19 with a line of 0.19 on CastSpeed |
| ChanceToBleed | inert | no attribute BleedChance; its mechanic is not built |
| ChanceToBlind | inert | no attribute BlindChance; its mechanic is not built |
| ChanceToBurn | inert | no attribute BurnChance; its mechanic is not built |
| ChanceToDoubleDamage | inert | no attribute DoubleDamageChance; its mechanic is not built |
| ChanceToFreeze | inert | no attribute FreezeChance; its mechanic is not built |
| ChanceToKnockback | inert | no attribute KnockbackChance; its mechanic is not built |
| ChanceToMarkOnHit | inert | no attribute MarkChance; its mechanic is not built |
| ChanceToPoison | inert | no attribute PoisonChance; its mechanic is not built |
| ChanceToShock | inert | no attribute ShockChance; its mechanic is not built |
| ChanceToSlow | inert | no attribute SlowChance; its mechanic is not built |
| ChanceToStun | inert | no attribute StunChance; its mechanic is not built |
| ChanceToWeaken | inert | no attribute WeakenChance; its mechanic is not built |
| ColdPenetration | working | a hit of its type on an enemy with 50% resistance: 50 without, 97 with a line of 0.47 on ColdPenetration; in play it changes nothing yet: every monster's resistances are 0 in DT_MonsterStats |
| ColdResistance | working | a hit of its type taken: 20 without, 17.8 with a line of 0.105 on ColdResistance |
| ColdResistanceShred | inert | no attribute ResistanceShred; its mechanic is not built |
| CooldownReduction | working | Acid Spit's cooldown: 4 without, 2.721 with a line of 0.47 on CooldownRecovery |
| CriticalChance | working | crits in 2000 seeded hits: 85 without, 477 with a line of 0.19 on CritChance |
| CriticalDamage | working | a crit's damage: 150 without, 197 with a line of 0.47 on CritDamage |
| CriticalStrikeAvoidance | inert | no attribute CritAvoidance; its mechanic is not built |
| CrowdControlResistance | working | seconds of a stun the user takes: 2 without, 1.78 with a line of 0.105 on CrowdControlResistance |
| DamageOverTimeMultiplier | working | a poison instance's total: 100 without, 111 with a line of 0.105 on DamageOverTimeMore |
| DamageReduction | working | a physical hit taken: 20 without, 16.8 with a line of 0.16 on DamageTakenMore |
| DamageToBleedingEnemies | inert | no attribute Damage; its mechanic is not built |
| DamageToCloseEnemies | inert | no attribute Damage; its mechanic is not built |
| DamageToDistantEnemies | inert | no attribute Damage; its mechanic is not built |
| DamageToEliteEnemies | inert | no attribute Damage; its mechanic is not built |
| DamageToFrozenEnemies | inert | no attribute Damage; its mechanic is not built |
| DamageToHealthyEnemies | inert | no attribute Damage; its mechanic is not built |
| DamageToInjuredEnemies | inert | no attribute Damage; its mechanic is not built |
| DamageToMarkedEnemies | inert | no attribute Damage; its mechanic is not built |
| DamageToSlowedEnemies | inert | no attribute Damage; its mechanic is not built |
| DamageToStunnedEnemies | inert | no attribute Damage; its mechanic is not built |
| DodgeEvasion | working | hits landed of 2000 seeded swings taken: 1899 without, 999 with a line of 0.47 on Dodge |
| FirePenetration | working | a hit of its type on an enemy with 50% resistance: 50 without, 97 with a line of 0.47 on FirePenetration; in play it changes nothing yet: every monster's resistances are 0 in DT_MonsterStats |
| FireResistance | working | a hit of its type taken: 20 without, 17.8 with a line of 0.105 on FireResistance |
| FireResistanceShred | inert | no attribute ResistanceShred; its mechanic is not built |
| FlatColdDamage | working | a hit of its type: 100 without, 147 with a line of 47 on ColdDamageFlat |
| FlatDamage | working | a physical hit's damage: 100 without, 147 with a line of 47 on DamageFlat |
| FlatDefense | working | a physical hit taken, with 50 flat Defense from elsewhere: 13.33 without, 11.05 with a line of 30.5 on Defense |
| FlatFireDamage | working | a hit of its type: 100 without, 147 with a line of 47 on FireDamageFlat |
| FlatHealth | working | maximum health: 1000 without, 1047 with a line of 47 on MaxHealth |
| FlatLightningDamage | working | a hit of its type: 100 without, 147 with a line of 47 on LightningDamageFlat |
| FlatMaximumMana | working | maximum mana: 100 without, 131 with a line of 30.5 on MaxMana |
| FlatPhysicalDamage | working | a physical hit's damage: 100 without, 147 with a line of 47 on PhysicalDamageFlat |
| FlatPoisonDamage | working | a hit of its type: 100 without, 147 with a line of 47 on PoisonDamageFlat |
| FreezeMagnitude | inert | no attribute FreezeMagnitude; its mechanic is not built |
| HealingReceived | working | health after a restore of 50 from 10: 60 without, 83.5 with a line of 0.47 on HealingReceived |
| HealthOnHit | working | health after one hit from 10: 10 without, 57 with a line of 47 on HealthOnHit |
| HealthOnKill | working | health after the web's kill handler from 10: 10 without, 57 with a line of 47 on HealthOnKill |
| HealthRegeneration | working | health gained in 2 s: 10 without, 28 with a line of 8.5 on HealthRegen |
| IncreasedColdDamage | working | a hit of its type: 100 without, 116 with a line of 0.16 on ColdDamageIncreased |
| IncreasedDamage | working | a physical hit's damage: 100 without, 116 with a line of 0.16 on DamageIncreased |
| IncreasedDefense | working | a physical hit taken, with 50 flat Defense from elsewhere: 13.33 without, 12.66 with a line of 0.16 on Defense; a player has no Defense of their own, so alone this line multiplies 0 |
| IncreasedFireDamage | working | a hit of its type: 100 without, 116 with a line of 0.16 on FireDamageIncreased |
| IncreasedHealth | working | maximum health: 1000 without, 1160 with a line of 0.16 on MaxHealth |
| IncreasedLightningDamage | working | a hit of its type: 100 without, 116 with a line of 0.16 on LightningDamageIncreased |
| IncreasedMaximumMana | working | maximum mana: 100 without, 116 with a line of 0.16 on MaxMana |
| IncreasedPhysicalDamage | working | a physical hit's damage: 100 without, 116 with a line of 0.16 on PhysicalDamageIncreased |
| IncreasedPoisonDamage | working | a hit of its type: 100 without, 116 with a line of 0.16 on PoisonDamageIncreased |
| LifeLeech | working | health after one hit from 10: 10 without, 57 with a line of 0.47 on Leech |
| LightningPenetration | working | a hit of its type on an enemy with 50% resistance: 50 without, 97 with a line of 0.47 on LightningPenetration; in play it changes nothing yet: every monster's resistances are 0 in DT_MonsterStats |
| LightningResistance | working | a hit of its type taken: 20 without, 17.8 with a line of 0.105 on LightningResistance |
| LightningResistanceShred | inert | no attribute ResistanceShred; its mechanic is not built |
| LowHealthDamageReduction | inert | no attribute DamageReduction; its mechanic is not built |
| ManaCostReduction | working | Acid Spit's mana cost: 10 without, 5.3 with a line of 0.47 on ManaCostLess |
| ManaOnHit | working | mana after one hit from 0: 0 without, 47 with a line of 47 on ManaOnHit |
| ManaOnKill | working | mana after the web's kill handler from 0: 0 without, 47 with a line of 47 on ManaOnKill |
| ManaRegeneration | working | mana gained in 2 s: 0 without, 18 with a line of 8.5 on ManaRegen |
| MeleeDamage | inert | no attribute Damage; its mechanic is not built |
| MinionDamage | working | a minion's outgoing damage scale: 1 without, 1.47 with a line of 0.47 on MinionDamageIncreased |
| MinionHealth | working | a minion's spawn life (FKitMinions::EvaluateHealth on the owner's line, as AFieldMonster spawns it): 100 without, 147 with a line of 0.47 on MinionHealthIncreased |
| MoreDamage | working | a physical hit's damage: 100 without, 111 with a line of 0.105 on DamageMore |
| MoreDefense | working | a physical hit taken, with 50 flat Defense from elsewhere: 13.33 without, 12.86 with a line of 0.105 on Defense; a player has no Defense of their own, so alone this line multiplies 0 |
| MoreHealth | working | maximum health: 1000 without, 1110 with a line of 0.105 on MaxHealth |
| MovementSpeed | working | the walk speed: 600 without, 666 with a line of 0.105 on MoveSpeed |
| PhysicalAsExtraCold | inert | no attribute ExtraDamage; its mechanic is not built |
| PhysicalAsExtraFire | inert | no attribute ExtraDamage; its mechanic is not built |
| PhysicalAsExtraLightning | inert | no attribute ExtraDamage; its mechanic is not built |
| PhysicalAsExtraPoison | inert | no attribute ExtraDamage; its mechanic is not built |
| PhysicalToColdConversion | inert | no attribute Conversion; its mechanic is not built |
| PhysicalToFireConversion | inert | no attribute Conversion; its mechanic is not built |
| PhysicalToLightningConversion | inert | no attribute Conversion; its mechanic is not built |
| PhysicalToPoisonConversion | inert | no attribute Conversion; its mechanic is not built |
| PoisonDamage | inert | no attribute PoisonDamage; its mechanic is not built |
| PoisonPenetration | working | a hit of its type on an enemy with 50% resistance: 50 without, 97 with a line of 0.47 on PoisonPenetration; in play it changes nothing yet: every monster's resistances are 0 in DT_MonsterStats |
| PoisonResistance | working | a hit of its type taken: 20 without, 17.8 with a line of 0.105 on PoisonResistance |
| PoisonResistanceShred | inert | no attribute ResistanceShred; its mechanic is not built |
| ProjectileSpeed | inert | no attribute ProjectileSpeed; its mechanic is not built |
| RangedDamage | inert | no attribute Damage; its mechanic is not built |
| ReducedCriticalDamageTaken | inert | no attribute CritDamageTaken; its mechanic is not built |
| ShockMagnitude | inert | no attribute ShockMagnitude; its mechanic is not built |
| SkillEffectDuration | working | Festering Shroud's buff seconds: 6 without, 7.14 with a line of 0.19 on SkillDuration |
| SlowMagnitude | inert | no attribute SlowMagnitude; its mechanic is not built |
| SpellDamage | inert | no attribute Damage; its mechanic is not built |
| StunDuration | working | seconds of a stun the user lands: 1 without, 1.19 with a line of 0.19 on StunDuration; a stun is cut by the hard-control clock (2 s here), so the line adds nothing to a stun already at it |
| ThornsReflect | working | health the enemy loses to one melee hit on the user: 0 without, 47 with a line of 47 on Thorns |
| WeakenMagnitude | inert | no attribute WeakenMagnitude; its mechanic is not built |

## Settled

- The list of lines and their three groups.
- One table, one row per line, editable by designers without code.

## Waiting on design

- The real value bands, the tier count and the tier weights (placeholders today), slot restrictions, local versus global (D-3.3); the prefix and suffix picks (D-5.4, eight coin-flips Jon's).
- Damage types (D-3.1).
- Six leftover questions from the review, each with a memo recommendation now (D-3.4): the name of the "mark" debuff, which enemy states get a damage line, whether minion lines stay, whether block exists without shields, what freeze magnitude means, and how lines reach a soul's moves.
- Accessories themselves, equipping them, rolling them and dropping them (E-3.2 to E-3.5).

## For engineers

`Source/SandboxARPG/Affixes/AffixRow.h` (`FAffixRow`, `EAffixCategory`, `EAffixOperation`). `Data/DT_Affixes.csv` is the import source, generated by `python scripts/gen_affix_tiers.py` from `Data/Authoring/AffixLines.csv` (107 lines: identity columns, `Group`, `Slot`, `T1Min`, `T1Max`, `Growth`, `Step`) and `Data/Authoring/AffixTiers.csv` (`Tier`, `WidthScale`), which also checks it (1070 rows, ten tiers per line, bands contiguous, ids unique, at least two families of each kind per colour; `--check` compares the file on disk with what the sheets generate); `Data/README.md` explains the columns and the import. The asset is `Content/Data/DT_Affixes`, imported by a human in the editor. Tags are plain names, not gameplay tags, which stay inside the ability wrapper.
