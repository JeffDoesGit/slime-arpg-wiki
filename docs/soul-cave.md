# The starter cave

*Built in roadmap items E-2.266 (a copy of the cave for each new player), E-2.267 (the sewer ladder and the cave's death rules), E-2.268 (the cave's monsters) and CT-2.72 (the starting forest removed). Provisional on the D-2.29 memo, sections 3, 5, 6 and 7 (register row D-2.29): the rules it touches, one open map per server (GDD 5.7) and an opening every new character plays alone, wait on a decision-log row. The level is a grey box until CT-2.71 dresses it from Epic's Soul: Cave pack.*

## What you see

A brand-new character, one the host has no saved record for, starts alone in a cave. Nobody else is in it: every new player gets a copy of their own, and two new players never meet there. You are the slime, with one attack, a short tackle on the left button (see [Souls](soul-model.md)).

The cave winds up the screen from a round room to a round den, "the Burrow":

1. **The wake**, a round room where you start. Its start point is a checkpoint. A Corpse sits against the wall beside the way out under an orange mark; click it and it gets up and attacks, and if you walk out of the room first it gets up anyway.
2. **The first pack.** A narrow passage of two bends opens into a wider cave where two weak Corpses stand together. The first of them you kill always drops a gem.
3. **The second pack.** A switchback of three bends opens into a second cave: a pair, and a third behind a pillar, far enough off that the pair does not pull it.
4. **The well.** Two more bends and a narrow throat lead into the Den. Just inside, on the left, stands a well under a warm light: walk into it to heal, fill your flasks and make it your checkpoint. From here a death brings you back to this well.
5. **The Den.** A Creepy, the cave's boss, stands in the middle and lets you look at it: it comes for you only once you are within about four and a half metres. Its kill always drops one whole Creepy soul at its body; walk onto it and it equips itself, and you are the Creepy from then on outside town. At the Den's far rim lies Walter, dead, with the human soul beside him; walk onto it and you wear it (cosmetic, it shows in town).
6. **The ladder**, in a nook off the Den's rim ("Sewer ladder"). It refuses a character with no soul yet and says "You cannot leave without a soul."; if you get one while standing on it, it lets you through within a second. It comes out at the sewer grate in town, west of the plaza well. The cave is gone behind you, and nothing leads back in.

The weak Corpses drop gems as any Normal does, but never a fragment, and never come back once killed. A death in the cave brings you back to the cave's start, or to the Den's well once you have walked into it. The death screen greys its town button there ("RETURN TO TOWN: NEEDS A SOUL"), and the server sends a town request from a cave to the start anyway. Inside the cave the minimap and its zoom buttons are hidden, since the map's picture holds nothing of a cave; the caption under it reads SOUL CAVE.

A character the host has a record for never goes back to the cave, whatever souls it holds (Jon, 2026-10-31: "no forced reset"): souls can be dropped and traded, and a character that has given every soul away plays on in town as the slime, tackling monsters until it wins another. It starts where it always did: where it was saved, or in town. A new player who quits inside the cave before getting a soul has been saved by then, so they come back in town as the slime, not in a fresh cave. Since the starting forest is gone (CT-2.72) the town is where a soulless character starts when the cave is switched off.

## How it works

The cave is a level of its own, `Content/Levels/Cave/Lvl_SoulCave`, a plain level (not World Partition), never placed in the one map (`CaveLevel` in `Config/DefaultGame.ini`). When a player joins with no saved record (asked of the persistence subsystem before the record is applied) and owning no non-cosmetic soul, the server loads a copy of it at runtime under a name of its own (`SoulCave_<slot>_<count>`), beside the one map at ground height: the first copy at X 200,000 and each next one 50,000 further along (`CaveOrigin`, `CaveSpacing`), lowest free slot first, with no limit on how many. A copy deep under the map does not work: the height fog blanks the view, and a spawner's navmesh query finds the town above it.

The server tells that player's machine, by one reliable message, to load the same name at the same place; a listen host's own world already holds its copy, so nothing is sent. Once the server holds the copy the player is teleported to its `CaveStart`, and the usual arrival hold (the one every teleport uses, [Zones](zones.md)) keeps a joiner still until their own machine reports the copy loaded. Every copy carries the same tags (`CaveStart`, `CaveDen`), so a teleport looks a tag up in the player's own copy first and never in someone else's. A spawner placed in the cave spawns its monsters into the copy, and those monsters are only ever sent to that copy's player. The copy is unloaded on the server and on the player's machine the moment the player arrives anywhere outside it, or logs out; copies whose player is gone are dropped.

The cave writes nothing to the saved character. There is no "cave done" flag: a character has done the cave once the host holds a record for it.

**The rooms.** `scripts/editor/make_soul_cave_greybox.py` lays the level from nothing each time it is run (pythonscript commandlet, editor closed, the level's lock held) as Jon's second shape of 2026-10-31, "the Burrow", from a table of segments at the top of the script: the wake a 16-sided room 2,000 uu across, passages 500 uu wide with mitred bends, two pack caves of 1,300 by 1,000 and 1,400 by 1,100, a throat 400 wide, the Den a 16-sided room 2,400 across, and the ladder's nook. Every floor is at the same height and uses the `Floor` collision preset, which the arrival hold and click-to-move need. A wall on the camera's side of its floor is low (200 uu) and fades only when it hides the player; every other wall is tall. A hidden lid over every room keeps the one map's sun out, so the cave is lit only by its own warm and cold lights. Then `scripts/editor/dress_soul_cave.py` dresses the grey box from Epic's Soul: Cave pack: the walls under the pack's rock, the floor cubes hidden, the pack's meshes placed as dressing, every grey cube's collision kept. A navmesh volume covers it all, and a `RegionZone` is named "Soul Cave".

**The Den's well** is a checkpoint teleport destination tagged `CaveDen`, with the town's well mesh as its marker and a warm light over it, placed 900 uu short of the boss so a player binds it before the boss can see them. `scripts/editor/place_cave_well.py` places it (the grey-box script places it too, so a fresh lay keeps it).

**The monsters.** `DA_Monster_Corpse_Cave` is a weak Corpse (36 life, damage scale 0.15, 12 experience, level 1) with no fragment chance. The sitting one in the wake is placed in the level asleep, with no spawner. The others come from the spawn rows `Cave_Pack` (two) and `Cave_Lone` (one), never refilled, and `Cave_FirstPack`, the first pack's row, which is `Cave_Pack` with `bFirstKillDropsGem` set: its spawner owes one gem, and the first of its monsters to die drops at least one gem whatever the drop roll says. A field monster remembers the spawner that made it (`SetHomeSpawner`, set before it finishes spawning), because once its AI controller possesses it, its owner is the controller and no longer the spawner. `DA_Monster_Creepy_CaveBoss` is an Elite at level 2 with 500 life, a damage scale of 0.05 and 65 experience, on the row `Cave_Boss`, with an aggro range of 450 uu. Its `DT_MonsterStats` row has `bGuaranteedSoul` set: its kill spawns one whole soul pickup at the corpse, owned by the monster so it lands in the monster's own copy. Its poison ticks do not follow the damage scale. Both assets are made by `scripts/editor/make_cave_monsters.py`. For proof runs, `Slime.KillSpawner <Spawner> [Count]` kills a spawner's monsters through the ordinary hit, so their drops run.

**The ladder** is a teleport point with `bRequiresSoul` set; it looks again once a second at a refused pawn still standing in it. Its destination, `SewerGrate`, is placed in town at (-600, 400) by `scripts/editor/place_sewer_grate.py`, with a flat grey slab as its marker that nothing collides with. Its place is the memo's suggestion and Jon's to move. The town's respawn destination (`TownDestination`) is `SewerGrate`.

## Waiting on design

- The decision-log row for GDD 5.7 and 7.4 (memo section 8): one open map per server, with a cave loaded for each new player.
- The cave's real look from Soul: Cave (CT-2.71).
- The boss's tuning against a slime with only the tackle.
- The well's body still reads dark under its lid.
- Where the grate stands in town, and its model.

## For testers

- `Slime.EnterCave [Pawn]` sends a pawn to a cave of its own; `Slime.LeaveCave [Pawn]` sends it to town and frees the copy; `Slime.ShowCaves` lists the copies on the host and, on any machine, the pawns that machine holds where the copies stand.
- `Slime.SoulCave 0` turns the cave off: a new character starts at the PlayerStart in town.
- `Slime.Respawn [town|checkpoint]` answers the death screen without the mouse.
- The ladder logs `refuses <pawn>: no soul yet` and `lets <pawn> through: it holds a soul now`; the boss's kill logs `Soul drop: ... dropped a whole DA_Soul_Creepy (guaranteed)`; a town respawn asked from a cave logs `asked for the town from its cave; back to the cave's start`.

## For engineers

`Source/SandboxARPG/Zones/CaveSubsystem.*` (`UCaveSubsystem`: `NeedsCave`, the slots, the load by `ULevelStreamingDynamic::LoadLevelInstanceBySoftObjectPtr`, the timer that sends the pawn in, the unload on arrival elsewhere or logout), `Zones/CaveConsoleCommands.cpp`, `Zones/Tests/CaveSubsystemTest.cpp`. Called from `AMonsterGameMode::HandleStartingNewPlayer` after the record is applied. On the pawn: `AMonsterCharacter::ClientLoadCave`, `ClientUnloadCave` (reliable) and `IsInCaveHere`, which the death screen and the minimap read. `UTeleportSubsystem::FindDestination` takes the pawn and prefers the pawn's own copy. `ATeleportPoint::bRequiresSoul` in `Zones/TeleportActors.*`; the guaranteed soul in `AFieldMonster::RollSoulFragment`. The soul pickup, teleport destination and teleport point bind their overlaps once, because a runtime level's actors begin play again.
