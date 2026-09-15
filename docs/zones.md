# Zones: the town gate and the safe zone

*Built in roadmap items E-2.41 (the gate) and E-2.9b (the safe zone). The gate is provisional on the human-soul memo (D-2.7); the safe zone is the settled rule GDD 6.1.*

## What you see

Outside the town stands a gate that will not let you through until you wear the soul lying on the ground nearby (the human soul, in the Worn slot of the inventory; a flag on the gate can relax this to owning it). Take it, wear it, touch the gate, and the door goes away for everyone on the server for the rest of the session.

Inside the town, or any place marked as safe, nothing can hurt you and you can hurt nothing. Press an ability key and nothing happens; a monster's swing does not land; a projectile that reaches you pops harmlessly. Monsters outside stop caring about you the moment you step in, and pick you up again when you step out. If you wear a soul (the human), you show its body while you are inside and your primary's body again once you step out (see [Souls](soul-model.md)).

## How it works, in plain words

**The gate** is one placed actor: a wall of collision every pawn hits, an invisible trigger in front of it, and a door mesh. When a player touches the trigger the server checks whether that player's character owns the gate's soul, in either slot or in the bag. The first owner opens it: the server flips one flag, the flag replicates, and every machine drops the wall and hides the door. There is no key item and no per-player state; an open gate is open.

**The safe zone** is a placed box. The server marks every pawn inside it, player or monster, and clears the mark when it leaves; a pawn standing in two overlapping zones stays marked until it has left both. The mark is one replicated flag on the pawn, and every combat rule reads that flag rather than asking the box:

- a move pressed while marked is refused on the server before anything plays, and the owning client refuses it locally too so no request goes out;
- a hit whose target or attacker is marked is not applied, with no flinch and no miss roll;
- a projectile that reaches a marked pawn pops without a hit;
- a monster never picks a marked pawn as its target and drops one that becomes marked.

Death cannot happen inside, so the death penalty never applies there; "no loss" follows from "no combat".

## Moving between zones

Since E-2.39 the world is one persistent level with each zone as a sublevel streamed in and out: the forest, the town, the first field. The forest, where everyone starts, is loaded before the first player appears, in the editor and in a standalone or packaged game alike (E-2.56); a joiner is told to load it the moment they connect. An exit is a box in one zone that names another zone and an entrance in it; walk in and the server loads the destination if nobody is there yet, moves you to the entrance standing still (a walk you had clicked ends the moment you touch the exit, so you cannot walk on into the next door, E-2.64; if you joined someone else's game, you then stay put, unable to move, until your own machine has the zone on screen, up to ten seconds, so you never fall through ground that has not loaded for you, E-2.57), and tells your machine to load that zone and drop the one you left. You see a black card, "Entering <zone>", until the destination has arrived; everyone else sees you go. Your bag, souls, gems, level and experience travel with you because they live on your character, not in the zone. A zone nobody stands in unloads on the server half a minute later. Each zone map is built at its own origin, so in the persistent level `Lvl_Slice` every sublevel gets its own world offset (the town sits 200 m north of the forest, the first field 400 m) and two loaded zones never overlap in space. The light rig lives in `Lvl_Slice` so every zone is lit whatever else is loaded. Navigation is generated at runtime when a zone loads (dynamic generation), so no zone map carries a baked navmesh of its own. Zones are shared: two players who take the same exit meet in the same place. Whether fields should instead be private copies per party is the open design call (D-8.6); the doors would not change. When the host of a game walks through an exit, the zone they leave stays loaded for anyone still standing in it; it unloads only once nobody has been there for 30 seconds (E-2.63).

## Settled

- GDD 6.1: cities are safe zones, no combat, no loss. The zone is the rule's whole implementation for now.
- The server validation list (DS-0.3 §3 row 12): every damage application is checked against the safe-zone rule on the authority.

## Waiting on design

- **What the gate's soul is** (D-2.7): the memo proposes a human soul worn as a cosmetic slot; the Corpse stands in.
- **Soul swapping restricted to cities** (D-4.2, roadmap E-2.12): the zone flag is ready for it, the check is not written.
- **Whether the town heals you, and what else it offers** (GDD 6.3).
- **Where the zone boundaries fall** once the town is its own level (D-8.6, E-2.39).

## For testers

- Place a `SafeZoneVolume` and size its Extent to cover the area. It never blocks movement.
- Inside it, an ability press logs `USoulComponent: <pawn> pressed '<move>' in a safe zone; refused` on the server.
- A monster losing you logs `<monster>: target lost`; a hit refused logs `Hit: <attacker> on <target> not applied (safe zone)`.
- `Slime.SetAttribute Health 0 <pawn>` still kills inside: it is a dev command, not damage.
- The gate logs `SoulGate <name>: opened by <pawn> (wears <soul>)` and `... does not wear ...; stays shut` (or owns / own with `bRequireWorn` off).

## For engineers

`Zones/ZoneSubsystem.*` (transitions, per-player streaming, unload timer; `Arrive` sends no streaming update to a local controller, because on a listen host the engine applies it to the server world), `Zones/ZoneActors.*` (`AZoneExit`, `AZoneEntrance`), the pawn's `CurrentZone` and `ClientZoneTransitionBegan`, the HUD's loading card. `Source/SandboxARPG/Zones/SoulGate.*` (the gate) and `Zones/SafeZoneVolume.*` (the zone). The flag lives on `Player/MonsterCharacter` (`IsInSafeZone`, `SetInSafeZone`, replicated push-model). The checks: `Souls/SoulComponent.cpp` (`ActivateMove`, `RequestActivateMove`), `Abilities/MonsterAbilitySystemComponent.cpp` (`ApplyHit`), `Abilities/MoveProjectile.cpp` (`HandleOverlap`), `Monsters/MonsterStateTree.cpp` (the sense evaluator). Nothing here replicates beyond the two flags.
