# Stats

*Built in roadmap item E-2.2 (pull request #29). Design rules: the affix pool's health, mana, defense and movement lines; GDD 7.7, changes happen on the server.*

## What they are

Your monster has eight numbers:

| Stat | What it means |
|---|---|
| Health, Max Health | How much you can take before dying, and the ceiling |
| Health Regen | Health recovered per second, once regeneration exists |
| Mana, Max Mana | The resource abilities will spend, and its ceiling |
| Mana Regen | Mana recovered per second, once regeneration exists |
| Defense | The one number that will reduce physical damage |
| Move Speed | How fast you move, once it is wired to movement |

Health and mana can never go below zero or above their maximum; the other numbers can never go below zero. Those guards are built in, so no later system can push a value out of range by accident.

## Where the numbers come from

Where a monster's base numbers come from is still an open design question (GDD 2.4): from the soul you wear, from your character, or both. The game now runs on the memo's recommendation (D-2.2), marked provisional until the decision lands: your character owns the base values and grows a little each level; every soul you have slotted, primary or secondary, adds its own growth on top. See [Level and growth](progression.md) for the formula and the placeholder numbers. Field monsters are different: their health, mana and defense are rows on the monster's own data, not derived.

[Gems](gems.md) sit on top: an equipped gem's lines modify the current value of a stat without touching the base, so a level-up and a gem never fight over the same number.

Defense, movement speed and regeneration start at zero on purpose. The memo gives them no base and no growth; they will come from accessories, gems and soul abilities. Nothing spends or regenerates yet either. Regeneration ticks, mana costs and movement speed all arrive with the systems that use them.

## Where you see them

The health and mana orbs on the [HUD](hud.md) show your current and maximum values. Everything else is visible only through the console commands below.

## For testers

Two host-only console commands:

- `Slime.SetAttribute Health 50` sets a stat on your monster. Add a pawn name at the end to set it on someone else's.
- `Slime.ShowAttributes` prints every monster's stats and whether that monster is the host's or a client's copy. Handy for telling two test windows apart.

## What is waiting on design

- Base values and growth (D-2.2): built on the memo, provisional until decided.
- Typed resistances, fire, cold, lightning, poison (D-3.1 memo recommends those four).
- Evasion (D-3.6 memo). The attacker-side numbers landed with E-3.9 on the memo's steps: flat, increased and more damage (untyped and per type), critical chance and damage, accuracy, a penetration per resisted type; they are read on the server only and not replicated, and a negative value is clamped to zero like every other attribute.

## For engineers

`Source/SandboxARPG/Abilities/MonsterAttributeSet.*`, `Abilities/AttributeConsoleCommands.cpp`. One `UAttributeSet` for the project, owned by `UMonsterAbilitySystemComponent` (the project's ability-component subclass, E-2.15), replicated as a registered subobject. Maxima are written by `UProgressionComponent` through `SetVitalMaxima` on the wrapper (E-2.8); the attribute set itself still ships every value at zero.
