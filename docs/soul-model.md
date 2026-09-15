# Souls

*Built in roadmap items E-2.3 and E-2.4 (pull request #20) and E-2.13 (pull request #27). Design rule: GDD 2.3, you equip one soul and become that monster.*

## The idea

A soul is a monster in item form. Equip it and you become that monster: its body, its animations and, once combat exists, its moves. This is the core of the game, and it is the first thing that works.

## What a soul holds

Each soul is one data asset a designer edits in the Unreal editor, no code involved:

- the monster's name and an icon for the inventory;
- its skeletal mesh, how it sits inside the collision capsule, and the capsule's size;
- three animation clips, idle, walk and run, with the speeds at which the game switches between them;
- the list of moves it grants (empty for now; see [Abilities](abilities.md)).

A soul carries no base stats. It carries two growth values, health and mana per character level, which apply while it sits in either slot; see [Level and growth](progression.md). Whether that is the final answer is the open question described under [Stats](attribute-set.md).

One soul exists: the **Creepy**, a creature from the Creatures Pack, with its idle, walk and run clips and a placeholder icon. It is the provisional starting soul until the design names one. The Creepy has no death animation, which combat will have to deal with.

## Equipping

You have two soul slots and a bag.

- The **primary slot** is the soul you are. Put a soul there and your body changes on every machine in the game.
- The **secondary slot** changes nothing you can see. Its one job so far is stat growth: the soul in it adds its per-level health and mana growth in full, the same as the primary ([Level and growth](progression.md)). The design memo behind it (D-4.2) says it will also open that soul's passive tree, once trees exist.
- The **bag** holds every soul you own but are not wearing. How many it can hold is still open.

You move souls between the bag and the slots by dragging them on the [inventory screen](inventory-shell.md). Behind the scenes, dragging sends a request to the server: "equip this soul". The server checks and decides; your screen updates when the answer comes back. This is the same path a multiplayer client uses, so single player and multiplayer behave identically.

The server also checks that you hold the soul you are equipping, in the bag or the other slot; a request for one you do not is refused and logged (since E-2.12). The design says swapping happens in cities only; that check exists behind a switch in the game settings, off for now, so you can swap anywhere until it is decided.

## Souls lying in the world

A soul can be placed in a level as a pickup (roadmap E-2.38). Walk over it and it goes into your bag; open the inventory to wear it. The server decides the pickup, so in multiplayer everyone sees it vanish at the same moment. It comes back after a delay (60 seconds by default, a placeholder) so each player in a session can take one; a pickup can also be set to vanish for good. If you already own that soul, walking over it does nothing and it stays for the others. A pickup carries a looping effect of its own so it can be seen without a mesh; the effect is chosen per pickup in the editor and stops while the soul is away.

You can also **wear** a soul without becoming it. A soul flagged cosmetic (the human, on the Sparrow body) goes in a third slot, Worn. Your moves and stats always stay the primary's. What changes is the body you show: inside a town or any safe zone you look like the worn soul, and outside one you look like your primary. With no primary equipped you look like the worn soul everywhere, and with nothing worn you look like the primary or the slime. Walking in and out of town swaps the body on every screen by itself, so you never take the human off to fight or the primary off to enter town (roadmap E-2.62, provisional on the D-2.7 memo). It cannot go in the primary or secondary slot, and nothing else can go in Worn. Outside a safe zone your primary's attack animations play on its own body; inside one no move fires anyway. The town gate opens only for a player wearing its soul, whichever body they happen to show.

The starting area uses this for its first soul, the one you find outside the town gate. What that soul is (the design asks for a human soul, which the design document does not yet allow) is an open design question, D-2.7. Until it is answered the pickup points at an existing monster soul.

The town gate is shut until someone who owns that soul touches it (roadmap E-2.41). A wall of collision and a door mesh block the way; the server checks the touching player's souls (either slot or the bag) and, on a match, opens the gate for everyone in the session and leaves it open. Other players see the door vanish at the same moment. A guard standing beside it is set dressing. The rule itself, who may enter a city and on what condition, is part of the same open question D-2.7; the memo recommends this shape and the code carries its marker.

## Fragments

Since E-2.7 field monsters drop pieces of their soul. When you land the killing blow on a Corpse or a Creepy there is a one-in-five chance (a placeholder row per monster, scaled by the host's Drop rates rule) that one fragment of that monster's soul goes to you. Fragments are a count, not an item: the bag shows a dimmed cell with "1/3" for a soul you are still assembling. On the third fragment the soul assembles by itself, wherever you are, and lands in the bag ready to wear. Fragments keep dropping after that, so a second copy assembles in time; what a duplicate soul is for is not decided. Bosses, when they exist, drop no fragments. Nothing here is saved between sessions.

## What is settled

- One equipped soul, and it is the monster you are (GDD 2.3).
- Souls are data assets (GDD 6.4).
- The server decides every equip (GDD 7.7 and the authority spec).

## What is waiting on design

- **How souls drop.** Built on the D-2.1 memo's recommendation (fragments, above) and marked provisional until decided; the memo also says bosses drop gear instead of souls, which changes a DECIDED rule and waits for Duilio.
- **The second slot's job** (D-4.2) and the skill tree it unlocks (D-8.2).
- **Soul upgrades** (D-4.1). **Stat growth from souls** (D-2.2) is built on the memo's recommendation and marked provisional until decided.
- **Carrying limits** and **safe-zone-only swapping**.

Nothing about souls is saved yet. Until the persistence design lands, the bag lives in memory for the session.

## For testers

`Slime.GrantSoul DA_Soul_Creepy` puts a soul in your bag (host only); `Slime.EquipSoul DA_Soul_Creepy` wears it without the inventory screen, through the same checks as the drag. Fragments: every kill that drops one logs `Fragment: <pawn> +1 DA_Soul_Corpse (1/3)`, the third `Soul assembled: <pawn> DA_Soul_Corpse`; raise `DropRates` in the server rules to make it near-certain.

## For engineers

`Source/SandboxARPG/Souls/SoulDefinition.*` (the asset), `Souls/SoulComponent.*` (slots including the worn one, bag, replication, server RPCs; `RefreshBody` picks the worn soul while the pawn's replicated `bInSafeZone` is set or no primary is equipped, else the primary, else the placeholder, and logs `SoulComponent: <pawn> body -> <soul> (safe zone|no primary|primary)`; the pawn's `OnRep_InSafeZone` calls it on every machine and the authority's setter calls the same function), `Souls/SoulConsoleCommands.cpp`, `Souls/SoulPickup.*` (placed pickup: server overlap, `GrantSoul`, replicated taken flag, respawn timer, an `IdleEffect` Niagara component set per instance and deactivated while taken), `Zones/SoulGate.*` (blocking box, trigger box, door mesh; server ownership check against `GateSoul`, one replicated open flag). Souls are Asset Manager primary assets of type `Soul` under `/Game/Souls`, found by name; locomotion is single-node, no animation blueprint.
