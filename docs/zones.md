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

Since E-2.39 the world is one persistent level with each zone as a sublevel streamed in and out: the forest, the town, the first field. The forest, where everyone starts, is loaded before the first player appears, in the editor and in a standalone or packaged game alike (E-2.56); a joiner is told to load it the moment they connect. An exit is a box in one zone that names another zone and an entrance in it; walk in and the server loads the destination if nobody is there yet, moves you to the entrance standing still (a walk you had clicked ends the moment you touch the exit, so you cannot walk on into the next door, E-2.64; if you joined someone else's game, you then stay put, unable to move, until your own machine has the zone on screen, up to ten seconds, so you never fall through ground that has not loaded for you, E-2.57), and tells your machine to load that zone and drop the one you left. If someone is already standing on the entrance, you land on a free spot a step or two around it instead; only when every spot is blocked are you set down on the entrance itself and the characters shuffle apart, so a crowd at a door never leaves you behind in the zone your machine has just dropped (E-2.67). You see a black card, "Entering <zone>", until the destination has arrived; everyone else sees you go. Your bag, souls, gems, level and experience travel with you because they live on your character, not in the zone. A zone nobody stands in unloads on the server half a minute later, with one exception: the zone players join into (the forest) stays loaded for the whole session (E-2.91). In the 2026-09-21 playtest the forest left the server 30 seconds after the last player walked into the town, and everyone who joined after that had no start point, appeared at the world's origin with nothing under them and fell; their own machine was also told not to load the forest, because a joiner's first list of zones is read from what the server has loaded. Each zone map is built at its own origin, so in the persistent level `Lvl_Slice` every sublevel gets its own world offset (the town sits 200 m north of the forest, the first field 400 m) and two loaded zones never overlap in space. The light rig lives in `Lvl_Slice` so every zone is lit whatever else is loaded. Navigation is generated at runtime when a zone loads (dynamic generation), so no zone map carries a baked navmesh of its own; until E-2.69 a second config line silently kept the old static mode, which is why joiners saw parts of the town they could not click-walk to (their fixed tile pool was sized for the forest alone) and the host saw "navmesh needs to be rebuilt". The pool is now sized for every zone at once, and the one navmesh actor lives in `Lvl_Slice`. Zones are shared: two players who take the same exit meet in the same place. Whether fields should instead be private copies per party is the open design call (D-8.6); the doors would not change. When the host of a game walks through an exit, the zone they leave stays loaded for anyone still standing in it; it unloads only once nobody has been there for 30 seconds (E-2.63).

**On the way to one map (E-2.76a).** Design has ruled that a server's world is one open map streamed around every player (GDD 5.7), and the doors above are the stepping stone. The first step is in the code and changes nothing you can see yet: the zone code now recognises a World Partition map. On such a map a door is only a teleport to the entrance it names, found by its tag wherever it stands. Nothing is loaded or unloaded for it and no zone is named to your machine, because the engine streams the ground around each player by itself, and the server keeps the whole map. The wait on arrival stays: if you joined someone else's game you still stand still until your own machine reports the ground under your feet, up to ten seconds. The start needs no special care either, since the engine never streams a player start away. On today's `Lvl_Slice` every line above still holds.

**The one map exists (CT-2.16).** `Content/Levels/Lvl_Slice_WP` is that World Partition map: the forest, the town and Zone1 on one landscape laid out as the design board draws them (the town at the centre, the forest to the south-west with its one-way gate into the town's south-west corner, Zone1 to the east; the west, south and north walls are kept for the next fields), one light rig, the same doors, gate, pickup, spawners and safe zone as the three zone maps, copied across with their per-actor edits. It was built new rather than converted, because the engine's convert commandlet stops on a landscape that lives in a sublevel and the old landscape held nothing worth keeping (`docs/plans/level-changes-2026-09-22.md`). It is not the map the game opens yet: the defaults still point at `Lvl_Slice`, and switching them, the cook and the packaged check are E-2.76b. On it the town and Zone1 are joined by open ground with no door between them: you walk out of the town's east arch, across a strip of grass, under Zone1's arch, and nothing loads, teleports or is logged. The forest keeps its one door, one-way into the town's south-west corner, through the E-2.76a path; what that door becomes is E-2.76c. Exit markers (CT-2.12) therefore show on this map only at the forest gate. The town keeps its cobble cube with its top 2 cm above the landscape until painted layers exist (CT-2.17); Zone1's grass cube is gone, the landscape is its floor.

## Settled

- GDD 6.1: cities are safe zones, no combat, no loss. The zone is the rule's whole implementation for now.
- The server validation list (DS-0.3 §3 row 12): every damage application is checked against the safe-zone rule on the authority.

## Waiting on design

- **What the gate's soul is** (D-2.7): the memo proposes a human soul worn as a cosmetic slot; the Corpse stands in.
- **Soul swapping restricted to cities** (D-4.2, roadmap E-2.12): the zone flag is ready for it, the check is not written.
- **Whether the town heals you, and what else it offers** (GDD 6.3).
- **Where the zone boundaries fall** once the town is its own level (D-8.6, E-2.39).

## Ground materials

Every floor in the slice wears one of two instances of a game-owned master material, `M_GroundTile` (`Content/Art/Materials/Ground/`). The master tiles in world space, through the pack's `MF_WorldCoords-XY`, so the forest landscape and a scaled floor cube show the same texture size without any UV work. It holds two texture sets, each one call of the material function `MF_GroundSet` (base colour, normal, an optional AORM map), and mixes them by a world-space noise. The parameters are `TileSize`, `TileScaleB` (set B tiles at a multiple of set A), `BlendThreshold`, `BlendContrast`, `BlendStrength`, `NoiseScale`, `Roughness` and `Tint`.

- `MI_Ground_Grass`: grass over dirt, dirt only in the noise peaks (threshold 0.68, contrast 6). On `Landscape_0` in StartingForest and the floor cube in Zone1.
- `MI_Ground_Cobble`: the same stone in both sets, set B at 1.6 times the size, blended softly, so no two tiles line up. On the floor cube in StartingTown.

A new biome is one more instance with its own two texture sets (CT-2.18, waiting on the biome list). The painted-layer landscape material for the one map calls the same `MF_GroundSet` per layer (CT-2.17). Every number is an eye placeholder; there is no design rule for the ground look.

For testers: a floor that shows the grey grid material means an instance failed to compile; the log line starts `Failed to compile Material Instance with Base M_GroundTile`. A texture with sRGB on cannot feed the AORM slot (`Sampler type is Linear Color, should be Color`).

## Exit markers

Every exit box now shows where it is and where it goes (CT-2.12). The `ZoneExit` actor carries two cosmetic components beside its trigger: a looping Niagara effect on the ground, picked per placed exit in the editor (the three slice exits use `NS_TeleportZone` from the RPGEnvironmentVFX pack, a ring of fire), and a floating text label that reads the exit's `DestinationZone` unless a `LabelText` override is typed in. The label's height, size, colour and rotation are properties on the exit; the rotation defaults to face the ARPG camera (pitch 55, yaw 135 against the camera's -55, -45), so the text reads upright from the play view. Nothing here replicates or ticks: the actor is placed in the map, so every machine has the components, and the effect loops on its own.

The three exits marked today: forest to town (`ZoneExit_ToTown_0`, label StartingTown), town to the first field (`ZoneExit_ToZone1_0`, label Zone1), first field back to town (`ZoneExit_1`, label StartingTown). In the two gates the walls hide most of the ring from the camera and the label does the work; in the open field both show.

This is a placeholder so testers find the doors. What an exit looks like in the game (a lit doorway, a signpost, a glowing gate, a banner on entry) is a design decision not taken; the label shows the internal zone name until zones have display names.

## For testers

- Place a `SafeZoneVolume` and size its Extent to cover the area. It never blocks movement.
- Inside it, an ability press logs `USoulComponent: <pawn> pressed '<move>' in a safe zone; refused` on the server.
- A monster losing you logs `<monster>: target lost`; a hit refused logs `Hit: <attacker> on <target> not applied (safe zone)`.
- `Slime.SetAttribute Health 0 <pawn>` still kills inside: it is a dev command, not damage.
- The gate logs `SoulGate <name>: opened by <pawn> (wears <soul>)` and `... does not wear ...; stays shut` (or owns / own with `bRequireWorn` off).
- The first time the forest would have unloaded, the host logs `ZoneSubsystem: StartingForest stays loaded (join zone); players join into it`. A join that logs `FindPlayerStart: ... NO PLAYERSTART` or `spawns before initial zone StartingForest is visible` is the old fault back.
- `Slime.Zone <ZoneName> [EntranceTag] [PawnName]` on the host moves a pawn to a zone's entrance through the same path as an exit (`Slime.Zone StartingTown FromForest`, `Slime.Zone Zone1 FromTown MonsterCharacter_1`). Dev builds only.
- An exit with no ring has no Niagara system set on its `Marker` component; an exit with no label has an empty `DestinationZone` and no `LabelText`. Both are set on the placed actor, under lock.

## For engineers

`Zones/ZoneSubsystem.*` (transitions, per-player streaming, unload timer; `SetJoinZone`, called by the game mode's `InitGame` with `InitialZone`, exempts that zone from `ScheduleUnloadIfEmpty` and `UnloadIfEmpty`, because `AGameModeBase::ReplicateStreamingStatus` builds a joiner's level list from the server's own flags and `FindPlayerStart` falls back to the WorldSettings actor at the origin; `Arrive` sends no streaming update to a local controller, because on a listen host the engine applies it to the server world; E-2.76a: `IsPartitionedWorld()` (`UWorld::IsPartitionedWorld`) switches `RequestTransition`, `PollPending`, `Arrive`, `FindEntrance` and `ScheduleUnloadIfEmpty` to the one-map path: `FindZone` is always null there, the entrance is found with a `TActorIterator<AZoneEntrance>` by tag (no untagged fallback), the pawn arrives on the next poll, no `ClientUpdateLevelStreamingStatus` is sent, and the hold waits on `UNetConnection::ClientHasInitializedLevel` for the level of the actor the Floor channel finds under the arrival spot (`FindGroundUnder`), the same engine report the sublevel path reads and the one the engine's own actor relevancy uses for cell actors; it assumes server streaming stays off (`wp.Runtime.EnableServerStreaming=0`, the 5.8 default). `IsZoneShownHere(Zone, Viewer)` is what the owning client's hold (`PollHeldZone`) and the loading card ask: the zone's streaming level in a sublevel world, `UWorldPartitionSubsystem::IsStreamingCompleted(Viewer)` in a partitioned one. `InitGame` skips the initial-zone flush and the join-zone pin on a partitioned map and says so in the log), `Zones/ZoneActors.*` (`AZoneExit`, `AZoneEntrance`), the pawn's `CurrentZone` and `ClientZoneTransitionBegan`, the HUD's loading card. `Source/SandboxARPG/Zones/SoulGate.*` (the gate) and `Zones/SafeZoneVolume.*` (the zone). The flag lives on `Player/MonsterCharacter` (`IsInSafeZone`, `SetInSafeZone`, replicated push-model). The checks: `Souls/SoulComponent.cpp` (`ActivateMove`, `RequestActivateMove`), `Abilities/MonsterAbilitySystemComponent.cpp` (`ApplyHit`), `Abilities/MoveProjectile.cpp` (`HandleOverlap`), `Monsters/MonsterStateTree.cpp` (the sense evaluator). Nothing here replicates beyond the two flags.
