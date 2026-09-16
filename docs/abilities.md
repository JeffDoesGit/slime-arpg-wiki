# Abilities

*Framework decided in D-1.2 (GAS-lite). The moves became playable, cosmetically, in roadmap items E-2.14a and E-2.20 to E-2.23 (pull requests #42 to #48); hits and damage arrived with combat (E-2.6, #56) and the numbers moved to a table (E-2.26, #59); mana costs, cooldowns and timed self-buffs are E-2.14; the Corpse's kit, the first with a charge, a planted mark and a cone at the cursor, is E-2.48; standing still for a move is E-2.71. Costs, cooldowns, both kits and the stand-still rule are marked provisional: no design rule names them yet.*

## Where this stands

You can press a key and your monster performs a move: it turns to face the cursor, plays the move's animation, and its effect appears in the world. Everyone in the game sees it. A melee move lands its hit a moment into the animation and a projectile hits on impact (see [Combat](combat.md)). A move can cost mana and can have a cooldown; both are checked by the server before anything happens. A move can also buff you for a few seconds.

## What you can do today

- **Four keys.** 1, 2, 3 and 4 each hold one move. Click a slot on the [HUD](hud.md) to pick which of your soul's moves sits there.
- **Aim with the mouse.** The move goes toward the cursor. Your monster turns to face it before the animation starts, on your machine as well as on the server's.
- **One move at a time.** A move runs to the end of its animation. Pressing another key during it does nothing; there is no queue and no cancel. A buff's animation is the exception: another key cuts it short and fires.
- **You stand still for a move.** Since E-2.71 a move stops your monster where it is and keeps it there until the animation ends; no more sliding through an attack. A click during the animation does nothing until it ends; a mouse button you keep holding moves you again the moment it does. Two exceptions: charges (Marrow Rush) carry you, and buffs (Festering Shroud, Bone Ward, any move with a buff duration) never stop you at all: their animation plays while you stand, and walking or pressing another move cuts it short while the buff itself stays on. `Slime.RootDuringMove 0` brings the old behaviour back for a session.
- **Attack standing still.** Holding Left Shift with a move key still works and now changes nothing, since every move stops you first.
- **Effects and projectiles.** A move can carry any number of visual effects, each placed on your body, at the cursor or on the target, and timed to the button press or to a marked frame of the animation. The Creepy's Acid Spit fires a projectile that flies toward the cursor and bursts on the first monster it meets.
- **Mana and cooldowns.** Some moves cost mana; pressing one you cannot afford does nothing, and the server logs why. Some moves have a cooldown; pressing one too soon does nothing. The [HUD](hud.md) slot shows the seconds left and greys the move you cannot afford.
- **Self-buffs.** A move can strengthen you for a fixed time. Festering Shroud gives Defense +15 for 6 seconds; Bone Ward gives Defense +15 and 4 health a second for 8 seconds. Both end by themselves.
- **Charges.** A move can carry you a set distance toward the cursor and hit everything you pass through, once each. Walls stop you. Marrow Rush is the first: 500 units in a quarter of a second. The developer command `Slime.Dash` tries it without a soul.
- **Marks.** Marrow Rush leaves a Splinter in every enemy it passes through. Charge through an enemy that already carries one and it shatters: everything within 250 units of that enemy takes a burst of damage, and the Splinter is spent. A Splinter fades after 6 seconds on its own.
- **Two hits from one press.** Grave Clutch claws twice, a moment apart, each a hit of its own.
- **Hits at the cursor.** Grave Hands lands on everything within 200 units of the point you aim at, instead of in the cone in front of your body, and reaches no farther than 400 units from you: aim beyond that and it lands at the edge.

The Creepy has four moves, named after its designed kit: **Venom Rake** (free, no cooldown), **Acid Spit** (10 mana, 4 s), **Bifurcating Rend** (15 mana, 6 s), **Festering Shroud** (25 mana, 12 s, the buff above). Their poison rules are still design.

The Corpse has four of its own since E-2.48: **Grave Clutch** (free, no cooldown, two claws of 8), **Marrow Rush** (10 mana, 3 s, the charge and the Splinter), **Grave Hands** (15 mana, 6 s, 12 damage in a cone at the cursor; its slow waits for the ailment rules), **Bone Ward** (25 mana, 12 s, the buff above). The field Corpse attacks you with Grave Clutch. Every one of those numbers is a placeholder in a table.

## How it works, in plain terms

Pressing a key sends the server one thing: "activate this move, aiming here". The server checks that the move belongs to the soul you are wearing, that you are not already mid-move, that the move is off cooldown and that you can pay for it. Then it takes the mana, starts the cooldown, turns your monster, runs the move, and tells every machine to play the animation and effects. Your own machine only did the turn early so the animation does not start facing the wrong way. The cooldown clock lives on the server and is sent to you, so the HUD never guesses.

Every number a move has, damage, cost, cooldown, buff, is one row of a data table (`DT_MoveStats`), not code. Retuning is a spreadsheet edit and a reimport. Every animation, effect, socket and colour is on the soul asset, editable by a designer.

## What is designed, pending the designer's sign-off

Memos on `main` propose the rules the first moves need:

- **Skill trees:** each soul has a small tree, and every node changes what a move does, never just a percentage (D-8.2). The first tree is the Creepy kit above.
- **Poison:** every hit that poisons adds a stack with its own timer (D-3.5). **Control effects** such as slow, freeze and stun: one instance at a time, the strongest wins (D-3.7).
- **Damage types:** physical, fire, cold, lightning, poison (D-3.1).
- **How a hit resolves:** a chance to hit, a chance to crit, then defense shaves damage off by a ratio that never reaches zero; getting hit plays an animation but does not interrupt you (D-3.6).
- **Monsters play by the same rules** as you (D-2.4).
- **Counted states:** a named count on a pawn that a move can add to, read and reset, with each stack fading on its own timer. Marrow Rush's Splinter is the first user (D-2.6); the console commands still drive it for testing.
- **The Corpse kit itself** (D-2.6): four moves that do not feed one another, two of them full damage plans on their own, one aura. Built as the memo recommends; the name and the fiction are the designer's.

## Known rough edges

- Firing a move from a client sometimes shows the monster facing a different direction on the server's screen than on the client's. Recorded as a defect.
- Mana climbs back at 2 a second (E-2.45), so a kit with costs is slow rather than dry between fights; `Slime.SetAttribute Mana 50` still refills it at once.
- Pressing Shift with a move key also triggers an engine debug shortcut in the editor. Harmless, being looked at.
- The Acid Spit projectile is still the placeholder dark colour; the poison-green recolour is content item CT-2.2.

## What comes next, in order

The Creepy kit's real rules (poison stacks, the Rend fork, the Shroud's contact poison); the ailment family, which also gives Grave Hands its slow.

## For engineers

`Source/SandboxARPG/Abilities/`: `MonsterAbility.*` (base class, name and server-side hook), `MonsterAbilitySystemComponent.*` (the project's ability-component subclass, wrapper type 1 of 4; counted states via `AddCount`, `GetCount`, `ResetCount`, one modifier-free duration effect per stack from `FMonsterEffects::BuildCountStackEffect`, server only), `CountConsoleCommands.cpp` (`Slime.AddCount <Id> <Cap> <Decay>`, `Slime.ShowCounts`, `Slime.ResetCount <Id>`, not in Shipping), `Souls/SoulConsoleCommands.cpp` (`Slime.UseMove <ClassName|Slot> [Pawn]` fires a worn move 300 uu ahead through `RequestActivateMove`, so a kit can be exercised from the console; `Slime.EquipSoul`), `MoveProjectile.*` (server-moved, replicated per contract C-3), `AnimNotify_SoulMoveEffect.*` (frame-timed effects), `Creepy/CreepyAbilities.*` (the four name-only classes), `Corpse/CorpseAbilities.*` (Grave Clutch, Grave Hands and Bone Ward name-only, driven by their rows; `UCorpseMarrowRush::OnDashCrossed` plants or shatters the Splinter through the counted-state API; the old `UCorpseSwordSlash` name resolves through a `ClassRedirects` line in `DefaultEngine.ini`). The move list, effects and projectile rows live on the soul asset (`FSoulMove`, `FMoveEffect` in `Souls/SoulDefinition.h`). Activation path: `USoulComponent::RequestActivateMove` to `ServerRequestActivateMove` to `ActivateMove` (moveset, dead, busy, cooldown and mana checks in that order; `UMonsterAbilitySystemComponent::SpendMana`; cooldown end written to the replicated `MoveCooldownEnds`, read back through `GetMoveCooldownRemaining` against the game state's server clock) to `UMonsterAbility::ActivateOnServer` (default: the row's timed buff via `ApplyTimedModifierSet`, a duration effect from `FMonsterEffects::BuildModifierEffect`) to `StartDash` when the row has a `DashDistance` (a `FRootMotionSource_MoveToForce` on CharacterMovement plus a crossing timer that lands the hit and then calls `UMonsterAbility::OnDashCrossed` on the move's class; the owning client applies the same source in `RequestActivateMove`; `Souls/DashConsoleCommands.cpp` for `Slime.Dash`) to `MulticastPlayMove`. A melee row's `ExtraHitDelays` add one `ResolveMeleeHit` timer each; a `bHitAtAim` row clamps the aim to the asset's `HitRange` from the pawn at activation and lands within the row's `AimRadius` of it, no arc. Rows: `FMoveStatRow` in `Abilities/MoveStats.h`, source `Data/DT_MoveStats.csv`. Keys come from `UHudLayoutSettings` in `Config/DefaultGame.ini`. Design detail: `docs/decisions/D-8.2.md`, `D-3.5.md`, `D-3.7.md`, `D-3.1.md`, `D-3.6.md`, `D-2.4.md`.
