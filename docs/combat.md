# Combat

*Built in roadmap item E-2.6 (branch `feat/combat-damage`). Provisional on four design memos: how a hit resolves (D-3.6), damage types (D-3.1), monsters on the player's rules (D-2.4) and player death (D-2.3).*

## What you see

Swing at a monster and, most of the time, it flinches and loses health. Sometimes you miss. Sometimes you crit and it loses more. Hit it enough and it plays its death animation and, a few seconds later, disappears. The monster does the same to you: its swings take your health down on the orb, and at zero you fall, then reappear at the start of the level with full health and mana a few seconds later. Acid Spit hurts whatever it bursts on.

Players cannot hurt each other. Monsters do not hurt monsters.

## How it works, in plain words

Every hit goes through one function on the server, the same one whether a player hits a monster or a monster hits a player. In order:

1. **Roll.** A chance to hit, 95% to begin with, clamped between a floor and a cap. A miss does nothing.
2. **Crit.** A chance to crit, 5% to begin with, for 150% damage.
3. **Type.** Every hit has exactly one type: physical, fire, cold, lightning or poison.
4. **Mitigate.** Physical damage is reduced by defense on a ratio that never reaches zero (defense 100 halves it). Elemental damage is reduced by that element's resistance, capped at 75%.
5. **Apply.** Health goes down. At zero, the pawn dies.

Before any of that, one check: a hit on or from a pawn standing in a safe zone is not applied at all, no roll and no flinch (GDD 6.1; see [Zones](zones.md)). Moves cannot even be pressed there.

All the numbers in that list are rows of one data table, `DT_CombatRules`. Every move's damage and type are a row in a second table, `DT_MoveStats`, named after the move; a monster's health, defense and resistances are its row in a third, `DT_MonsterStats`. Only a move's reach and timing stay on the soul asset. Nothing is in code, and retuning a fight is a spreadsheet edit and a reimport, no build.

The first pass at those numbers (E-2.26) aimed for the Corpse dying in about five Venom Rake hits and a fresh level-1 player dying in about nine swings. The first Zone1 fight (2026-09-13) found two Creepy monsters unkillable by a level-1 Creepy, so the Creepy kit hits half again as hard (Venom Rake 30), a Creepy monster has 40 health, and a monster row now carries a **damage scale**: the monster hits with its soul's moves at that fraction of a player's damage (the Creepy monster at a fifth, the Corpse at full). Two Creepies swing every 1.3 seconds each, so at half damage they still won in nine seconds against a level-1 Corpse. One column, read off the attacker; the damage function does not ask who is hitting. They are placeholders with no design behind them, marked as such, and will move as soon as someone plays with them.

A melee move lands its hit a fixed fraction of a second into its animation, on every enemy within a short range and a cone in front of the attacker. That timing is a number on the move, not a mark on the animation, so a dedicated server that plays no animations still lands hits. A projectile lands its hit on whatever it bursts on.

Being hit plays a flinch animation, and only that: it never interrupts what you were doing. Dying plays the death animation and holds it. A monster despawns after a delay; a player respawns at the level's PlayerStart with full vitals, on the same character, so nothing in the bag or the slots is lost.

## Settled

- The server resolves every hit; your machine shows the result after the fact (GDD 7.7, 9.8).
- One function for both directions; the differences between a monster and a player are numbers on their assets, never a separate rule.
- Numbers live in `DT_CombatRules` and on the soul and monster assets.

## Waiting on design

- **All of it, formally.** The four memos are recommendations; Duilio has not decided. If a decision changes the function, the code changes and the data does not.
- **Ailments and poison** (D-3.5): not built; the Creepy kit's poison arrives with its real abilities.
- **The XP penalty on death** (D-2.3): in since E-2.31 as a 10% placeholder of your progress toward the next level; see [Progression](progression.md).
- **The city as the respawn point**: the level's PlayerStart stands in until the city exists.
- **Accuracy, evasion, crit and reduction lines** from accessories: fixed at their base values until the affix system applies them.
- **Damage numbers and miss text on screen**: allowed, not built.
- **The Creepy's death**: it has no death clip in its pack, so it holds its pose.
- **Hit stop** (E-2.72): a landed hit freezes the attacker and the victim for the move's `HitStopSeconds` (a few hundredths of a second, capped at 0.12) and shakes the attacker's own camera. A victim mid-swing keeps its swing and only freezes; one outside a swing plays its hit-react clip and freezes on it. Cosmetic, after the server's hit, off with `Slime.AttackFeel 0`.

## For testers

Host only: `Slime.SetAttribute Defense 100 FieldMonster_0` halves the next physical hit on it. Damage lines print in the server log as `Hit: <attacker> hit <target> for <n> <type>`. `Slime.SetAttribute Health 0 <pawn>` still kills outright.

## For engineers

`Source/SandboxARPG/Abilities/`: `DamageTypes.h` (`EDamageType`, `FHitSpec`, `FHitOutcome`), `CombatRules.*` (`DT_CombatRules` lookup, path in `DefaultGame.ini`), `MonsterAbilitySystemComponent::ApplyHit` (the function), the four resistance attributes on `MonsterAttributeSet`. `Souls/SoulComponent.*`: `ResolveMeleeHit` by timer, `ReactToHit`, `PlayDeath`, `Revive`, one `MulticastPlayReaction`. `Souls/SoulDefinition.h`: the hit shape rows on `FSoulMove` and the two reaction clips. `Abilities/MoveStats.*` (`DT_MoveStats` row per ability class: `FlatDamage`, `DamageType`) and `Monsters/MonsterStats.*` (`DT_MonsterStats` row per monster asset name), both with config paths in `DefaultGame.ini` (E-2.26). `Abilities/MoveProjectile.*` carries an `FHitSpec`. `Player/MonsterCharacter.*`: `IsHostileTo`, replicated `bDead`, `HandleHealthDepleted` and `Respawn`; `Monsters/FieldMonster.*` overrides both. Provisional markers: registry rows D-3.6, D-3.1, D-2.3, D-2.4 and "hit shape and per-move damage".
