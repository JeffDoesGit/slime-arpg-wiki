# The monster you play

*Built in roadmap item E-2.1, extended by every item since. Design rule: GDD 1.1, the player is a monster.*

## What you see

You start as a green sphere. That is not a monster, it is a stand-in: no rule in the design document says what monster a new player begins as, so rather than invent one, the game shows an obvious placeholder until you equip a soul. Once you do, the sphere becomes that monster, with its own idle, walk and run animations.

Movement is click-to-move, like the Unreal top-down template the project started from, tightened up: your monster turns to face a new direction at once and starts and stops almost instantly, so a reversal never has it running the wrong way for a beat. The camera hangs well back on a fixed diagonal, looking down at about 55 degrees through a narrow lens, so the ground reads flat and you see a good stretch of it in every direction, the way an action RPG is usually framed; it never turns with your monster. Those numbers are placeholders until a design rule sets them, and the developer command `Slime.Camera` lets us try others in play without a rebuild. Press `I` to open the inventory. 1, 2, 3 and 4 perform the moves on your [HUD](hud.md) bar; they hit nothing yet (see [Abilities](abilities.md)).

A new character starts with an empty bag. The first soul lies on the ground outside the town gate of the starting forest; walk over it and it is yours to equip (see [Souls](soul-model.md)). A config list can still seed the bag for testing, and whether the slime ever starts with a soul is a design decision still open.

## How it works, in plain terms

Your monster is one Unreal character that carries three things:

1. **An ability system.** The engine's Gameplay Ability System, the same framework Fortnite uses, in a slimmed-down form the team calls GAS-lite, reached through the project's own thin wrapper. It holds your stats and runs your moves. Nothing predicts on your machine; every move is resolved on the server.
2. **Your stats.** See [Stats](attribute-set.md).
3. **Your soul slots and bag.** See [Souls](soul-model.md). When a soul is equipped, this is what swaps your body.

In multiplayer, each player's monster exists on the server and is mirrored to every client. Your machine only ever sends "I clicked here", "I pressed this move aiming there" or "I chose this from a menu". Clients path-find on their own machine for click-to-move, as the host always did; two fixes in this batch made that true for clients and stopped the navigation mesh rebuilding on every map load.

## What is settled

- The player is a monster (GDD 1.1).
- Server-authoritative: the server resolves everything (GDD 7.7).
- The networking driver (Iris) and the ability framework (GAS-lite) are decided.

## What is waiting on design

- **The starting monster.** Sphere until then.
- **Stats and where they come from.** Everything is zero; a memo (D-2.2) proposes character base values plus growth from each equipped soul.
- **Movement feel.** The instant facing and the acceleration values are placeholders, registered as provisional; no design rule sets movement tuning.
- **Damage and death.** Moves play but do not hurt. Combat is the next code item after the field monster.

## For engineers

`Source/SandboxARPG/Player/MonsterCharacter.*`, `Player/MonsterGameMode.*`, `Config/DefaultGame.ini` (`DefaultBagSouls`; movement values sit in the pawn constructor for now). The ability component is the project subclass `UMonsterAbilitySystemComponent` (E-2.15); no engine GAS type is named outside `Abilities/`, per CLAUDE.md 6.7. Navigation settings: `Config/DefaultEngine.ini` (E-2.19, E-2.24).
