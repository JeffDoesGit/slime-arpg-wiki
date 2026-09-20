# The monster you play

*Built in roadmap item E-2.1, extended by every item since. Design rule: GDD 1.1, the player is a monster.*

## What you see

You start as a green sphere. That is not a monster, it is a stand-in: no rule in the design document says what monster a new player begins as, so rather than invent one, the game shows an obvious placeholder until you equip a soul. Once you do, the sphere becomes that monster, with its own idle, walk and run animations.

Movement is click-to-move, like the Unreal top-down template the project started from, tightened up: your monster turns to face a new direction at once and starts and stops almost instantly, so a reversal never has it running the wrong way for a beat. The camera hangs well back on a fixed diagonal, looking down at about 55 degrees through a narrow lens, so the ground reads flat and you see a good stretch of it in every direction, the way an action RPG is usually framed; it never turns with your monster. Those numbers are placeholders until a design rule sets them, and the developer command `Slime.Camera` lets us try others in play without a rebuild. Press `I` to open the inventory. 1, 2, 3 and 4 perform the moves on your [HUD](hud.md) bar; they hit nothing yet (see [Abilities](abilities.md)).

The cursor only ever lands on the ground (since E-2.54). Click on a tree canopy, a wall or a bush and your monster walks to the ground under it; aim a move across a bush and it lands on the ground beyond. Before this, anything the camera could see caught the click, so a canopy stopped you dead and a wall took your aim, and each offending prop had to be fixed by hand. A monster under the cursor is the one thing that still counts: aim a move at it and the move goes to the monster, not the ground behind it.

You can also walk with the keyboard or a pad (since E-2.52): W A S D and the arrow keys move you along the camera's diagonal, so W is always "up the screen", and the left stick does the same; the pad's face buttons fire the four bar slots. Press a movement key while a click-path is running and the path is dropped. Click-to-move stays as it was. Since S walks backward, the skill tree opens on P instead (see [The HUD](hud.md)).

What stands between the camera and you no longer hides you (since E-2.55). A tree crown or the town wall that covers your monster becomes a faint dark stipple while it does, and is solid again the moment you step out. It reads as "something is here" rather than as a shape of its own: hiding the mesh outright was far too strong, and a translucent ghost read as mist under a crown because hundreds of leaf cards stack, so the ghost is a dithered mask whose screen pattern every layer shares. Bushes are the exception: they have no collision since the cursor pass, so the sweep never sees them and a monster inside one is hidden. Polish of the whole treatment, including drawing your monster's silhouette through occluders the way Diablo does, is E-2.81.

In multiplayer every machine now paints every player's sphere the same colour (since E-2.79): the server picks one per player by arrival order, green, blue, orange, magenta, and replicates it. Before this the tint was applied on each machine at spawn, before a client's mesh had its material, which is why the 2026-09-15 playtest saw brown on one screen and green on another. The colour is only on the sphere; a worn soul's body is the same for everyone.

A new character starts with an empty bag. The first soul lies on the ground outside the town gate of the starting forest; walk over it and it is yours to equip (see [Souls](soul-model.md)). A config list can still seed the bag for testing, and whether the slime ever starts with a soul is a design decision still open.

Everyone sees the same brightness (since E-2.88). The camera pins its exposure to one value instead of letting the engine adapt it per view, because a joiner's window adapted differently from the host's and washed the Creepy from orange to olive. The value is the one the free auto exposure had settled on in the host's window, tuned against a joiner's window; it lives in config, and `Slime.Exposure <EV100>` tries another for the session (`auto` frees it again). One cost: no adaptation between a bright field and a dark interior, so a dark place later needs its own value.

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
- **Movement feel.** The instant facing and the acceleration values are placeholders, registered as provisional; no design rule sets movement tuning. The movement keys and the pad buttons are placeholders too; rebinding is E-2.53.
- **What an occluder should look like.** The dither threshold and tint are ini rows; whether the game wants a silhouette, a textured fade or something else is open (E-2.81).
- **Damage and death.** Moves play but do not hurt. Combat is the next code item after the field monster.

## For engineers

`Source/SandboxARPG/Player/MonsterCharacter.*`, `Player/MonsterGameMode.*`, `Config/DefaultGame.ini` (`DefaultBagSouls`; movement values sit in the pawn constructor for now). The ability component is the project subclass `UMonsterAbilitySystemComponent` (E-2.15); no engine GAS type is named outside `Abilities/`, per CLAUDE.md 6.7. Navigation settings: `Config/DefaultEngine.ini` (E-2.19, E-2.24). Cursor traces (E-2.54): one project trace channel `Floor` (`ECC_GameTraceChannel1`, `#define ECC_Floor` in `SandboxARPG.h`), default response Ignore, declared in `DefaultEngine.ini` `[/Script/Engine.CollisionProfile]` beside a `Floor` collision preset (BlockAll plus Floor) that the ground actors carry: the forest Landscape and the `Floor_Placeholder` cubes of the town and Zone1. Everything else ignores the channel with no per-asset edit, because the engine's BlockAll preset gives a new custom channel its own default. Click-to-move in `BP_TopDownController` traces Floor through its `Get Location Under Cursor` function; the ability aim in `AMonsterCharacter::PressAbilityKey` traces Floor, then a Pawn object-type trace, and a live hostile `AMonsterCharacter` under the cursor wins as the aim (its actor location); `aim: floor|monster <name>|forward` at Verbose on the press, never per frame. Keyboard and pad movement (E-2.52): legacy axis mappings `MoveForward` and `MoveRight` in `Config/DefaultInput.ini` (W S Up Down and the left stick Y; D A Right Left and the left stick X), bound in `SetupPlayerInputComponent` to `MoveForward` and `MoveRight`, which add movement input along the camera boom's yaw frame and call `StopMovement` on the controller when a path is running; the pad button of each slot is a `PadKey` on the `AbilitySlots` rows. Occlusion fade (E-2.55): `Player/OcclusionFadeComponent.*` on the pawn, ticking only where it is locally controlled (`RefreshEnabled` from `PossessedBy` and `OnRep_PlayerState`), one `SweepMultiByObjectType` sphere from the camera to the pawn per frame, WorldStatic and WorldDynamic, no per-frame allocation; a hit component whose material has the `OcclusionFadeParameter` scalar gets a dynamic instance at `OcclusionFadeOpacity`, otherwise every slot swaps to one shared dynamic instance of `OcclusionGhostMaterial` (`Content/Art/Materials/M_OcclusionGhost`: Masked, Unlit, Two Sided, `DitherTemporalAA` with the `Opacity` parameter as threshold into Opacity Mask, `Color` on Emissive), and with no ghost material the mesh hides when `bOcclusionHideWhenNoParameter`; Floor-preset components and Landscape actors are skipped; all rows in `UHudLayoutSettings` "Occlusion Fade". Placeholder colour (E-2.79): `PlaceholderPalette` on the pawn, picked in `PossessedBy` on the authority by the player controller's index, `PlaceholderColor` replicated push-model with `OnRep_PlaceholderColor` calling `ApplyPlaceholderColor`.
