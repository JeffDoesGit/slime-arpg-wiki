# Gems

*Built in roadmap item E-3.2. Design rules: GDD 3.1 to 3.3, equipment is a few slots carrying rolled stat lines. Provisional on the slot memo (D-3.2), which makes gems the equipment layer.*

## The idea

Your equipment is gems. There are no armour or weapon slots and no separate accessory that a gem plugs into: the gem is the item, and it carries the stat lines. You have nine slots, two red, two green, two blue, two that take any colour, and one for special gems. A red gem goes in a red slot or an any slot; a special gem goes only in the special slot. Colour says what the gem carries: red gems hold offense lines, blue gems control and utility, green gems defense, one colour per category of the [affix table](affix-table.md). A gem that breaks that rule is refused when it is created. That mapping is Jon's reading of the design board, written into the slot memo and marked provisional until Duilio confirms it; what the special slot holds is still open.

A gem has a colour, a tier, a rarity name, and a list of lines. The tier runs from I to X and names the gem's stone: a red gem is Red Jasper at tier I, Pyrope Garnet at IV and Ruby at X; blue runs Sodalite to Sapphire, green Aventurine to Emerald, iridescent Common Opal to Black Opal, special Quartz to Diamond. Ten tiers, the stone names and which stones belong to the iridescent and special colours are Jon's placeholders, marked provisional until Duilio records them (E-3.7). Today the tier changes only what the gem looks like and is called; nothing about its lines depends on it yet, and every gem a monster drops is tier I. Each line names one of the 107 entries in the [affix table](affix-table.md) and the value it rolled. Since E-3.3 a gem can be rolled. The roll takes the gem's colour, a rarity and the gem's tier. Rarity says how many lines: one, two or three (R1, R2, R3, placeholder names). For each line the server picks a line family of the colour's category that the gem does not already carry, then one of that family's ten tiers, never above the gem's own tier and leaning toward it, then a number inside that tier's band. So a Ruby X can carry a tier 10 line worth 174 to 227 health-sized points where a Red Jasper I carries 5 to 10, and an R1 gem of a high tier can beat an R3 gem of a low tier on its one line. The same seed always gives the same gem. Every number, the ten tiers and the weights are placeholders for Duilio's balancing pass.

## What happens when you equip one

The server moves the gem from your bag into the slot and applies every line to your stats. Each line's table row says which stat it targets and how:

- **Flat** adds points. Fifty flat maximum health is fifty more.
- **Increased** adds a percentage. Every increased line on the same stat is summed first, then applied once, so two lines of ten percent are twenty percent, not twenty-one.
- **More** multiplies on its own. Two lines of ten percent more are twenty-one percent.

Unequip and the lines come off. Modifiers sit on top of your base stats, so [level and soul growth](progression.md) keep working underneath: a level-up changes the base, the gem's percentage still applies to the new number.

Lines whose target stat does not exist yet (most of the table until damage and resistance attributes land) are logged and skipped, never applied. Nothing is persisted; contract C-4 carries gems later.

## What you see

Your gems sit in the bag on the [inventory screen](inventory-shell.md) as their stones, each with its tier numeral in the corner of the cell; hover for the stone's name, the numeral and the lines ("Pyrope Garnet IV"). A slotted gem and a gem being dragged show the same stone. A gem whose colour and tier have no stone shows the plain colour icon instead. Drag one onto a slot of its colour to equip it (an iridescent gem, a fifth colour with no lines decided yet, goes only in the two iridescent slots, and the other four colours never do, since E-3.6); drag it back to the bag to take it off. The health orb moves when the lines land. Since E-3.5.

## Drops

Since E-2.32 a field monster can drop a gem when it dies: half the time at the host's default drop rate, a small tinted sphere at the corpse that anyone can walk over. The first player to touch it gets it in their bag; it vanishes for everyone, and an untaken one fades after two minutes. Each drop is one colour and one fixed line, from a list of five: a red one with flat damage (which does nothing yet), blue ones with maximum mana or mana regeneration, green ones with health or defense. With `Slime.AffixRoll` on (the default since E-3.3) a drop is rolled instead: a colour, a rarity by weight, always tier I for now, and its lines from the [affix table](affix-table.md); `Slime.AffixRoll 0` brings the five fixed lines back. What drops which tier waits on the real drop table (E-3.4). The host's Drop rates rule scales the chance.

## For testers

Host-only commands, except the equip pair which a client may run and the server then decides:

- `Slime.GrantGem Green FlatHealth_T01=50 IncreasedHealth_T01=0.1` puts a green gem with two defense lines in your bag. A number after the colour is the tier, 1 to 10, tier 1 without one: `Slime.GrantGem Red 10` is a Ruby. Add a pawn name before the lines to give it to someone else. Line ids are `DT_Affixes` row names, a line and its tier; a red gem with a health line is refused.
- `Slime.RollGem Green R3 42 10` rolls a green three-line gem of tier X with seed 42 and puts it in your bag; the first number is the seed (random without one), the second the tier (1 without one). Run it twice and the two gems match line for line.
- `Slime.EquipGem 0 0` equips bag gem 0 into slot 0. A colour the slot refuses is logged and rejected.
- `Slime.UnequipGem 0` sends it back to the bag.
- `Slime.ShowGems` prints every pawn's bag and slots as this machine sees them; `Slime.ShowAttributes` shows the result.

## What is waiting on design

- The slot decision itself (D-3.2): count, colours, what a colour permits, what the special slot holds.
- The real value bands, tier count and weights behind the roll, prefix and suffix, slot restrictions (D-3.3); the roll runs on placeholders until then.
- Rarity: three placeholder rows that set the line count; names, counts and drop weights are Duilio's.
- Gem tiers: whether there are ten, what the stones are called, and what a tier does to a gem's lines and to what drops (D-3.3 section 8; the roll is E-3.3, the drop E-3.4).
- Bag capacity for gems, as for souls.

## For engineers

Drops: `Equipment/GemDrops.*` (the roll), `Equipment/GemPickup.*` (the actor), `Equipment/GemDropSettings.*` and the `GemDropSettings` section of `DefaultGame.ini` (chance, lifetime, lines). Called from `Monsters/FieldMonster.cpp` `Die()`.


Stones: `Equipment/GemBaseTable.*` reads `DT_GemBases` (fifty rows named `<Colour>_<NN>`: colour, tier, display name, soft icon) through the `GemBaseTable` config path; `UGemBaseTable::ApplyBase` sets a gem's tier and base id on the server, `UInventoryLayoutSettings::FindGemIcon(const FGemInstance&)` loads the stone when a cell refreshes and falls back to the per-colour `GemIcons` row.

`Source/SandboxARPG/Equipment/`: `GemInstance.h` (`FRolledAffix`, `FGemInstance` with its `Tier` and `BaseId`, `FGemList` FastArray), `GemEquipmentComponent.*` (bag, slots, `SlotAccepts`, request RPCs, modifier bookkeeping), `GemConsoleCommands.cpp`. `Affixes/AffixTable.*` reads `DT_Affixes` by row name through a config path. `Abilities/MonsterEffects.*` is the wrapper's fourth type: it builds one infinite `GameplayEffect` from a list of `FAttributeModifier`, and `UMonsterAbilitySystemComponent::ApplyModifierSet` / `RemoveModifierSet` hold the active-effect handles, so no GAS type leaves `Abilities/`.
