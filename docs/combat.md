# Combat

*Built in roadmap item E-2.6 (branch `feat/combat-damage`). Provisional on four design memos: how a hit resolves (D-3.6), damage types (D-3.1), monsters on the player's rules (D-2.4) and player death (D-2.3).*

## What you see

Swing at a monster and, most of the time, it flinches and loses health. Sometimes you miss. Sometimes you crit and it loses more. Hit it enough and it plays its death animation and, a few seconds later, disappears. The monster does the same to you: its swings take your health down on the orb, and at zero you fall, then reappear at the start of the level with full health and mana a few seconds later. Acid Spit hurts whatever it bursts on.

Players cannot hurt each other. Monsters do not hurt monsters.

## How it works, in plain words

Every hit goes through one function on the server, the same one whether a player hits a monster or a monster hits a player. In order:

1. **Roll.** A chance to hit, 95% to begin with, times one plus the attacker's accuracy lines, clamped between a floor and a cap. A miss does nothing.
2. **Crit.** A chance to crit, 5% plus the attacker's critical-chance lines, for 150% damage plus its critical-damage lines.
3. **Type.** Every hit has exactly one type: physical, fire, cold, lightning or poison. The number is the move's damage plus the attacker's flat lines (untyped, and of the hit's type), times its increased lines summed, times its more lines compounded (since E-3.9; every red gem was inert before it).
4. **Mitigate.** Physical damage is reduced by defense on a ratio that never reaches zero (defense 100 halves it). Elemental damage is reduced by that element's resistance, capped at 75%, less the attacker's penetration of that type, never below zero.
5. **Apply.** Health goes down. At zero, the pawn dies.

Before any of that, one check: a hit on or from a pawn standing in a safe zone is not applied at all, no roll and no flinch (GDD 6.1; see [Zones](zones.md)). Moves cannot even be pressed there.

All the numbers in that list are rows of one data table, `DT_CombatRules`. Every move's damage and type are a row in a second table, `DT_MoveStats`, named after the move; a monster's health, defense and resistances are its row in a third, `DT_MonsterStats`. Only a move's reach and timing stay on the soul asset. Nothing is in code, and retuning a fight is a spreadsheet edit and a reimport, no build.

The first pass at those numbers (E-2.26) aimed for the Corpse dying in about five Venom Rake hits and a fresh level-1 player dying in about nine swings. The first Zone1 fight (2026-09-13) found two Creepy monsters unkillable by a level-1 Creepy, so the Creepy kit hits half again as hard (Venom Rake 30), a Creepy monster has 40 health, and a monster row now carries a **damage scale**: the monster hits with its soul's moves at that fraction of a player's damage (the Creepy monster at a fifth, the Corpse at full). Two Creepies swing every 1.3 seconds each, so at half damage they still won in nine seconds against a level-1 Corpse. One column, read off the attacker; the damage function does not ask who is hitting. They are placeholders with no design behind them, marked as such, and will move as soon as someone plays with them.

A melee move lands its hit a fixed fraction of a second into its animation, on every enemy within a short range and a cone in front of the attacker. That timing is a number on the move, not a mark on the animation, so a dedicated server that plays no animations still lands hits. A projectile lands its hit on whatever it bursts on.

Being hit plays a flinch animation, and only that: it never interrupts what you were doing. Dying plays the death animation and holds it. A monster despawns after a delay; a player respawns at the level's PlayerStart with full vitals, on the same character, so nothing in the bag or the slots is lost.

## Control ailments

Since the tree-verbs pass of 2026-09-27 (E-2.123, E-2.124) a landed hit can also leave a **control ailment** on its target: a slow, a stun, a knockback or a weaken. A move's row names which one and how often (Grave Hands slows every enemy it hits), and the tree can apply one too (Reeking Shroud weakens everyone standing in the shroud). Each kind is one instance per target: a second application refreshes the clock to the full duration and keeps the stronger of the two magnitudes, so nothing stacks (a refresh never shortens: a shorter application on a longer one leaves the time that was left, since 2026-10-04); different kinds coexist. A **slow** cuts the walk speed and the swing speed by its percentage. A **stun** stops the pawn: it cannot move, cannot act, and the move it was in the middle of is cut short, its unlanded claws dropped; it is the one thing that interrupts a move. A **knockback** shoves the target a fixed distance away from the attacker in one hop; it has no duration. A **weaken** cuts the damage the target deals by its percentage. A Boss-tier monster ignores stun and knockback and takes slow and weaken at half strength. Your own tree can push back: Sure Footed and Heavy Bones reject a knockback and halve a slow or a stun; Molt clears every control on you when it triggers. The name plate shows what is on a pawn and how long it has left, read from the server. Since E-2.127 three more kinds exist on the same terms: a **freeze** stops the pawn where it stands and it cannot act, for a base time times the freeze's strength (two seconds at 1.0), but unlike a stun it does not cut short the move already in progress; a **shock** makes the pawn take that much more damage from every hit, counted after armour and resistance; a **blind** takes that much off the pawn's accuracy, so its swings miss more (never below the 5 % floor). A boss ignores a freeze as it ignores a stun, and takes shock and blind at half strength. **Crowd-control resistance** shortens every control on its bearer by its value, up to 75 %, never to nothing, and never touches a knockback; the Crowd-control resistance gem line gives it. The chance-to-freeze, shock and blind gem lines do nothing yet: no rule says how a gem's chance adds to a move's. Every number is a `DT_CombatRules` row and every rule is the D-3.7 memo's, which Duilio has not decided.

## Settled

- The server resolves every hit; your machine shows the result after the fact (GDD 7.7, 9.8).
- One function for both directions; the differences between a monster and a player are numbers on their assets, never a separate rule.
- Numbers live in `DT_CombatRules` and on the soul and monster assets.

## Waiting on design

- **All of it, formally.** The four memos are recommendations; Duilio has not decided. If a decision changes the function, the code changes and the data does not.
- **Poison** (D-3.5): built as stacking instances since E-2.109; see [Progression](progression.md). **Control ailments** (D-3.7): the four the trees need since E-2.123, provisional on the memo; freeze, shock, blind and crowd-control resistance wait (E-2.127).
- **The XP penalty on death** (D-2.3): in since E-2.31 as a 10% placeholder of your progress toward the next level; see [Progression](progression.md).
- **The city as the respawn point**: the level's PlayerStart stands in until the city exists.
- **Accuracy, evasion, crit and reduction lines** from accessories: fixed at their base values until the affix system applies them.
- **Damage numbers and miss text on screen**: allowed, not built.
- **The Creepy's death**: it has no death clip in its pack, so it holds its pose.
- **Hit stop** (E-2.72): a landed hit freezes the attacker and the victim for the move's `HitStopSeconds` (a few hundredths of a second, capped at 0.12) and shakes the attacker's own camera. A victim mid-swing keeps its swing and only freezes; one outside a swing plays its hit-react clip and freezes on it. Cosmetic, after the server's hit, off with `Slime.AttackFeel 0`.

## For testers

Host only: `Slime.SetAttribute Defense 100 FieldMonster_0` halves the next physical hit on it. Damage lines print in the server log as `Hit: <attacker> hit <target> for <n> <type>`. `Slime.SetAttribute Health 0 <pawn>` still kills outright. `Slime.ApplyControl stun` (or `slow`, `knockback`, `weaken`, with an optional magnitude, duration and pawn name) lands one on a pawn through the same path a hit takes; `Slime.ShowControls` and `Slime.ClearControls` read and end them; the log says `Control: <pawn> gained <kind> ...`.

## For engineers

`Source/SandboxARPG/Abilities/`: `DamageTypes.h` (`EDamageType`, `FHitSpec`, `FHitOutcome`), `CombatRules.*` (`DT_CombatRules` lookup, path in `DefaultGame.ini`), `MonsterAbilitySystemComponent::ApplyHit` (the function), the four resistance attributes on `MonsterAttributeSet`. `Souls/SoulComponent.*`: `ResolveMeleeHit` by timer, `ReactToHit`, `PlayDeath`, `Revive`, one `MulticastPlayReaction`. `Souls/SoulDefinition.h`: the hit shape rows on `FSoulMove` and the two reaction clips. `Abilities/MoveStats.*` (`DT_MoveStats` row per ability class: `FlatDamage`, `DamageType`) and `Monsters/MonsterStats.*` (`DT_MonsterStats` row per monster asset name), both with config paths in `DefaultGame.ini` (E-2.26). `Abilities/MoveProjectile.*` carries an `FHitSpec`. `Player/MonsterCharacter.*`: `IsHostileTo`, replicated `bDead`, `HandleHealthDepleted` and `Respawn`; `Monsters/FieldMonster.*` overrides both. Provisional markers: registry rows D-3.6, D-3.1, D-2.3, D-2.4 and "hit shape and per-move damage".

## Hit shapes (E-2.148)

A melee move hits whatever its shape on the ground touches. There are three shapes: a cone (a wedge in front of the attacker), a circle (a disc in front of or around the attacker) and a capsule (a straight band in front, round at its far end). Each move has its own shape and sizes in the move table, so a wide sweep, a short jab and a ground slam can each be tuned on their own row. A target is hit when its body touches the shape, not only when its centre is inside. Players and monsters use the same shapes through the same code, and both walk to the distance their own shape reaches before they swing.

A move whose row sets no shape uses the reach and arc on the soul asset, as every move did before, so the moves in the game today land as they did.

For testers: `Slime.ShowHitShapes 2` draws every melee hit's shape on the ground for two seconds, green when it touched an enemy and red when it touched nobody. It is drawn where the server tests the hit, so you see it when you host or play alone, not as a joiner.

For engineers: `FHitShape` in `Abilities/HitShape.h` (`FromRow`, `Touches`, `EdgeReach`); the columns `HitShape`, `ShapeLength`, `ShapeRadius`, `ShapeAngle`, `ShapeOffset` on `FMoveStatRow`; `USoulComponent::ResolveMeleeHit` is the one caller of the test. Lengths count from the edge of the attacker's capsule, a circle's radius from its centre. The numbers are placeholders.

## Chains of triggered effects (E-2.166)

Some effects set off other effects: a hit plants a poison stack, a shatter throws Splinters onto neighbours, a kill by poison spreads the poison. Left alone, two of these can feed each other for ever. One rule stops that. Every hit, counted stack and placed area carries an **origin** and a **chain depth**.

- **Direct**: what a player or monster did on purpose. A swing, a projectile, a dash crossing, a placed area, an aura, the shatter of Marrow Rush or Tear.
- **Triggered**: what another effect set off. A poison tick, a Hive Heart debt payment, a Bone Choir shatter, the Splinters that Shrapnel and Ossuary plant.
- **Expiry**: what happens when something runs out. Brittle Bone's shatter.

A reaction that makes a further hit, stack or area fires only when its cause is Direct and the chain is shorter than the bound (3). So a retaliation answers a swing and never a poison tick, and a choir shatter does not plant Shrapnel that could set off another shatter. Three reactions are held to the bound only, because the plain rule would break what they are for: Carrion's spread when a poison tick kills, Brittle Bone on a Splinter that Shrapnel planted, and Hive Heart deferring a triggered hit. Kill credit, experience, fragments, flask charges and the heal or mana a kill gives are never asked.

Two outcomes differ from before the rule: a Bone Choir shatter plants no Shrapnel, and Carrion's copies of copies stop at depth 3.

For testers: `Slime.TriggerOriginRule 0` turns the rule off and the game behaves as it did before; `Slime.MaxHitChainDepth` sets the bound. A reaction refused by the bound warns once per source pawn in the server log; a reaction refused because its cause was not Direct says nothing.

For engineers: `EHitOrigin`, `FHitSpec::Origin`, `ChainDepth` and `MarkTriggered` in `Abilities/DamageTypes.h`; the pure rules `UMonsterAbilitySystemComponent::EvaluateTriggerAllowed` and `EvaluateChainDepthAllowed`, asked through `IsTriggerAllowed` and `IsChainDepthAllowed` at every reaction site (`Progression/SoulTreeHooks.cpp`, `Abilities/Corpse/CorpseAbilities.cpp`, `Abilities/MoveArea.cpp`, `Souls/SoulComponent.cpp`). Tests `SandboxARPG.Combat.TriggerOrigin.*`. Provisional: register row `trigger origin`.

## When a hit's reactions run (E-2.167)

A hit can set three things off on its victim: the answer to the blow itself (the flinch, Caustic Blood, Ossuary), the answer to where the blow left its health (Rigor, Molt), and the death. These used to run wherever the code happened to reach them, two of them in the middle of writing the new health. Molt could heal while the hit was still being counted, and the hit then reported a negative number.

Now the hit finishes first. It writes health, fills in its result and logs its `Hit:` line. Then its reactions run, always in the same order: the answer to the blow, then the answer to the health, then the death. The death still tells the killer's tree (Marrow Tithe, Grave Dust, Rot Feeder, Carrion) before the victim is paid out and cleared, as before. All of it happens before the hit hands its result back, so whoever dealt the hit sees a victim that is already dead when the hit killed.

If a reaction makes another hit on the same pawn, that hit finishes completely, with its own reactions, before the first hit's remaining reactions go on. A change of health that no hit made (regeneration, a flask, a heal, a dev command) is told at once, as before. Unfinished and Hive Heart shape a hit before it lands and are not part of this.

What differs from before: a hit that pushes a pawn under Molt's threshold reports its own damage, and the heal follows it; on a killing hit the answer to the blow runs before the death, where it ran after.

For testers: `Slime.DeferHitReactions 0` puts every reaction back where it ran before.

For engineers: `FHitReactionQueue` in `Abilities/HitReactionQueue.h` (`BeginHit`, `Push`, `EndHitAndDrain`, `DrainOrder`), held by `UMonsterAbilitySystemComponent`; `ApplyHit` opens a frame after its early refusals, `HandleHealthChanged` records while a frame is open, `FinishHit` is the one end of `ApplyHit` and runs the frame. One array reused for every frame, nothing allocated per hit once warm. Tests `SandboxARPG.Combat.HookQueue.*`. Provisional: register row `hook queue`; the order is this build's, the research note names none.
