# Gems

*Built in roadmap item E-3.2. Design rules: GDD 3.1 to 3.3, equipment is a few slots carrying rolled stat lines. Provisional on the slot memo (D-3.2), which makes gems the equipment layer.*

## The idea

Your equipment is gems. There are no armour or weapon slots and no separate accessory that a gem plugs into: the gem is the item, and it carries the stat lines. You have nine slots, two red, two green, two blue, two that take any colour, and one for special gems. A red gem goes in a red slot or an any slot; a special gem goes only in the special slot. Colour says what the gem carries: red gems hold offense lines, blue gems control and utility, green gems defense, one colour per category of the [affix table](affix-table.md). A gem that breaks that rule is refused when it is created. That mapping is Jon's reading of the design board, written into the slot memo and marked provisional until Duilio confirms it; what the special slot holds is still open.

A gem has a colour, a rarity name, and a list of lines. Each line names one of the 107 entries in the [affix table](affix-table.md) and the value it rolled. Rolling does not exist yet (E-3.3), so today a line's value is whatever the developer command gave it.

## What happens when you equip one

The server moves the gem from your bag into the slot and applies every line to your stats. Each line's table row says which stat it targets and how:

- **Flat** adds points. Fifty flat maximum health is fifty more.
- **Increased** adds a percentage. Every increased line on the same stat is summed first, then applied once, so two lines of ten percent are twenty percent, not twenty-one.
- **More** multiplies on its own. Two lines of ten percent more are twenty-one percent.

Unequip and the lines come off. Modifiers sit on top of your base stats, so [level and soul growth](progression.md) keep working underneath: a level-up changes the base, the gem's percentage still applies to the new number.

Lines whose target stat does not exist yet (most of the table until damage and resistance attributes land) are logged and skipped, never applied. Nothing is persisted; contract C-4 carries gems later.

## What you see

Your gems sit in the bag on the [inventory screen](inventory-shell.md) as colour swatches; hover for the lines. Drag one onto a slot of its colour, or an any slot, to equip it; drag it back to the bag to take it off. The health orb moves when the lines land. Since E-3.5.

## Drops

Since E-2.32 a field monster can drop a gem when it dies: half the time at the host's default drop rate, a small tinted sphere at the corpse that anyone can walk over. The first player to touch it gets it in their bag; it vanishes for everyone, and an untaken one fades after two minutes. Each drop is one colour and one fixed line, from a list of five: a red one with flat damage (which does nothing yet), blue ones with maximum mana or mana regeneration, green ones with health or defense. There is no rolling, no tiers and no rarity: those wait on the affix-range decision and the real drop table (E-3.3, E-3.4). The host's Drop rates rule scales the chance.

## For testers

Host-only commands, except the equip pair which a client may run and the server then decides:

- `Slime.GrantGem Green FlatHealth=50 IncreasedHealth=0.1` puts a green gem with two defense lines in your bag. Add a pawn name before the lines to give it to someone else. Line ids are `DT_Affixes` row names; a red gem with a health line is refused.
- `Slime.EquipGem 0 0` equips bag gem 0 into slot 0. A colour the slot refuses is logged and rejected.
- `Slime.UnequipGem 0` sends it back to the bag.
- `Slime.ShowGems` prints every pawn's bag and slots as this machine sees them; `Slime.ShowAttributes` shows the result.

## What is waiting on design

- The slot decision itself (D-3.2): count, colours, what a colour permits, what the special slot holds.
- Value ranges, tiers, prefix and suffix, slot restrictions (D-3.3), which unblock rolling (E-3.3).
- Rarity: a name on the gem with no effect until tiers are decided.
- Bag capacity for gems, as for souls.

## For engineers

Drops: `Equipment/GemDrops.*` (the roll), `Equipment/GemPickup.*` (the actor), `Equipment/GemDropSettings.*` and the `GemDropSettings` section of `DefaultGame.ini` (chance, lifetime, lines). Called from `Monsters/FieldMonster.cpp` `Die()`.


`Source/SandboxARPG/Equipment/`: `GemInstance.h` (`FRolledAffix`, `FGemInstance`, `FGemList` FastArray), `GemEquipmentComponent.*` (bag, slots, `SlotAccepts`, request RPCs, modifier bookkeeping), `GemConsoleCommands.cpp`. `Affixes/AffixTable.*` reads `DT_Affixes` by row name through a config path. `Abilities/MonsterEffects.*` is the wrapper's fourth type: it builds one infinite `GameplayEffect` from a list of `FAttributeModifier`, and `UMonsterAbilitySystemComponent::ApplyModifierSet` / `RemoveModifierSet` hold the active-effect handles, so no GAS type leaves `Abilities/`.
