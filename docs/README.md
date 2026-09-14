# Systems

What exists in the game right now, one page per system, in plain language. Each page says what the thing does, what you see when you play, what is settled and what is still waiting on a design call. Pages are updated whenever the system changes, in the same change that touched it (ROADMAP §1.11). Adding a page to the table below adds it to the wiki's navigation.

This is the wiki the team browses: https://randy-lahey.github.io/slime-arpg-wiki/ (offline since the repository went private on 2026-09-10; these pages are the source, and roadmap item E-0.11 restores the site)

Playing together from the editor, host and join by console, the Sunday run sheet: [docs/playtest.md](../playtest.md).

| System | What it is | State |
|---|---|---|
| [The monster you play](player-pawn.md) | Your character in the world: how it moves, what it looks like before and after wearing a soul | playable, placeholder look |
| [Stats](attribute-set.md) | Health, mana, defense, speed: the numbers on your monster | health and mana derived from level and souls; defense and speed still zero |
| [Level and growth](progression.md) | Your character level, experience, regeneration, and the skill tree you spend points in | level, XP and regen in place; tree screen and allocation in place, numeric nodes work, verb nodes display only; all provisional on the stats and progression decisions |
| [Souls](soul-model.md) | A monster's soul as an item: what it holds, how you equip it, the bag you carry them in, souls placed in the world | playable with two souls; placed pickups |
| [Inventory screen](inventory-shell.md) | The screen you open with `I`: soul slots, gem slots, the bag | playable for souls; gems drawn only |
| [Abilities](abilities.md) | The moves a soul gives you | playable: keys, animations, effects, a projectile, damage through combat, mana costs and cooldowns; ailments and trees still design |
| [The HUD](hud.md) | Health and mana orbs and the ability bar with cooldown and mana state | playable, placeholder layout |
| [Affix table](affix-table.md) | The 107 stat lines an accessory can roll, as data | table exists; gems read it; no items roll yet |
| [Gems](gems.md) | The equipment item: colour, rarity, rolled lines; nine typed slots; lines applied to your stats | in place by console, no screen; provisional on the slot decision |
| [Zones](zones.md) | The town gate that opens for a soul's owner, and the safe zone where no combat happens | gate provisional on the human-soul memo; safe zone in place, GDD 6.1 |
| [Field monsters](field-monsters.md) | The monsters in the field: what they are, how they think, where they come from | two types as data; server-only brain; placed spawners read a table and the host's Mob density rule |
| [Server rules](server-rules.md) | The twelve rules a host sets for their server, where they live, and what happens when one is typed wrong | in place; values are placeholders until tuned |
| [Replication](replication.md) | Which networking driver the game uses, the one-line fallback, and the test that proved both work | in place; per-player filtering owed (S-2.3) |

## Things that are true everywhere

- **The server is the referee.** Nothing you do on your machine decides an outcome. You press a button, the server decides what happened, and you see the result. This holds in single player too, where your machine is also the server.
- **Design decides, code follows.** A rule that is still open in the design document is not built. When we build on an assumption anyway, the code is marked *provisional* and the assumption is listed in a register so it can be revisited the moment the decision lands.
- **Numbers live in data, not code.** Anything a designer might want to tune is in a data asset, a table or a config file.
- **Developer commands** start with `Slime.` and only work on the host.
- **Souls** are data assets named `DA_Soul_<Monster>` in the `Content/Souls` folder.
- **Art** is kept in two mirrored places: source files under `Art Assets/`, imported versions under `Content/Art/`.
