# The stash

*Built in roadmap item E-3.30 (2026-11-19), in two pieces: the stash itself and its save (piece A), then the chest, the screen and Ctrl and a left click (piece B). Design rule: GDD 6.2, decided 2026-11-19 on Jon's order and cards: "there is a chest in town next to gemcrafting bench, implement a player stash that will survive between saves and different playtest sessions. should use same ui style as bench and soul keeper but should just have rows of items. ctrl left click to move items from player inventory to stash and vice versa." The size (80 cells) and the chest's reach (250 uu) are placeholders under register row `stash size`.*

## What you see

A chest stands in the town beside the gem bench. Move the cursor onto it and it lights with the bench's gold rim, and its name label, "Stash", brightens. Click it and your character walks to it and the stash opens; standing beside it, the pick-up key opens it too.

The stash opens as a panel on the right of the view in the gem bench's and the soul-keeper's look, with your inventory beside it on its left, as the inventory stands beside the bench. The panel has:

1. **The title**, "STASH", and the close button in its top right corner.
2. **A hint**: "Ctrl + click an item to move it between your bag and the stash."
3. **One sunk well of 80 cells**, 8 rows of 10, the bag's cells exactly: soul copies first (the soul's orb, its stage in gold, the copies from two), then fragment rows (the dimmed orb with "2/3"), then bench item stacks (the Etching Needle, the Expanding Lattice and the Polishing Wheel with their counts), then gems (the stone, its tier and its rarity frame). Every tooltip is the bag's.
4. **A count** under the well: the cells in use of the stash's 80, "12 / 80".

**Moving things.** While the stash is open, hold Ctrl and left click:

- a cell in your bag, and what that cell holds goes into the stash: a gem; every copy of that soul that is in your bag (a copy in a soul slot stays where it is); a soul's fragment row whole; a bench item's whole stack;
- a cell in the stash, and what it holds comes back to your bag the same way, whole.

That is the only way anything moves. Nothing drags into or out of the stash, and a right click on a stash cell does nothing. With the stash shut, Ctrl and a left click on a bag cell does what a plain click does. A soul or gem in a slot or socket is never moved: put it back in the bag first.

A cell in the bag that holds a soul's copies and its fragments toward another copy moves the copies; the fragments stay in the bag on a cell of their own, and a second Ctrl and click moves them.

**What refuses.** A full stash refuses a deposit that needs a new cell and tells you "Stash full"; a deposit into a row the stash already has (more copies of a soul it holds, more of an item stack it holds) always fits. A full bag refuses a gem, and a stack it has no cell for, and tells you "Inventory full"; a stack that would pass the bag stack's cap (50) stays whole in the stash with the same notice. Souls and fragments never fill the bag. A stashed fragment row that comes back and completes a copy assembles that copy, as picking the fragments up would.

**When it closes.** By its X, by the inventory's X, by Escape, by the inventory key, by opening the gem bench, the soul-keeper, the tree or the options, and when you walk away: the bench's close range, 400 uu from the chest. The server also refuses every move made more than the chest's reach away, so a stash never moves anything from across the town.

**Whose it is.** Yours: the stash is kept on your account on the server you play on, not on the chest and not on your character. Another player at the same chest sees their own stash, never yours. It is saved with your account every time your character is, and it is still there after a quit, a new build or another session on that server.

## How it is built

The stash's contents live on a component on the player's pawn (`UStashComponent`), four lists: gems, soul copy rows, fragment rows and bench item rows. One gem is one cell; one row of a soul's copies, of its fragments or of one bench item is one cell. The client only ever asks for a move ("deposit this gem", "withdraw the Creepy copies", "withdraw the needles") through one server call; the server re-reads its own state, checks every rule and the chest's reach, and moves the thing or refuses. The lists then reach that player's machine alone, and the screen redraws from them.

The chest (`AStashActor`) is a placed actor with a mesh, a reach and the bench's hover and label. It holds nothing. The screen (`UStashScreen`) is built in code like every screen in the game. Its cells are the bag's own cell classes with every mouse handler but one taken away, so they draw exactly as the bag's do and can start no drag, lift nothing onto the cursor and take no drop.

The save is the account record's Stash section (record version 6). An account saved before the stash existed (version 5) loads with an empty stash.

## For engineers

`Source/SandboxARPG/Stash/StashComponent.*` (the lists, the pure rules `EvaluateUsedCells`, `EvaluateStashHasRoom`, `EvaluateDeposit`, `EvaluateWithdraw`, `EvaluateAddToRow`, `EvaluateTakeRow`, the moves in `ApplyMove`, the requests through `ServerRequestStashMove`, rate-limited on its own row in `Server/ServerRules.cpp`), `Zones/StashActor.*` (reach, hover, label), `UI/StashScreen.*` (the screen, its three cell subclasses `UStashSoulCell`, `UStashGemCell`, `UStashItemCell`, the pure `BuildCells` and `EvaluateCtrlPress`, and `HandleBagCtrlPress`, which `USoulItemCell`, `UGemItemCell` and `UItemStackCell` call first in their press handler), `UI/MonsterHUD.*` (`ToggleStash`, `IsStashOpen`, `GetStashScreen`, the hover pass's `HoveredStash`, the inventory docked beside it, the mutual exclusion with the bench, the keeper, the tree and the options), `Player/MonsterCharacter.*` (`FindStashUnderCursor`, `FindStashInReach`, `BeginStashWalk`, `UpdateStashWalk`, the pick-up key), `Server/Persistence/CharacterRecord.*` (`FAccountRecordStash`, `FSandboxRecordVersion::AccountStash`, the version-5 read branch in `UAccountRecord::Serialize`), `Server/Persistence/PersistenceSubsystem.cpp` (restore in `ApplyToPawn`, save in `WriteRecord`).

Moves go through the bag's own functions: `UGemEquipmentComponent::TakeGemById` and `GrantGem`, `USoulComponent::TakeBagCopies`, `AddSoulCopy`, `TakeFragments` and `AddFragment` (at the monster row's count, so a completed row assembles), `UCraftingComponent::SpendCount` and `GrantItems` (the stack cap and a free bag cell for a new stack). A deposit puts the thing into the stash before it takes it from the bag, so the bag's own save event writes a state with it in one place.

The bench items are the three ids `FCraftingRules::IsBenchItem` names, the counts the bag draws as cells (`UInventoryScreen::RefreshSlots`). Shards and essence are counts on the same component that the bag shows as numbers, not cells; they never go in the stash.

Placing the chest: put an `AStashActor` (`/Script/SandboxARPG.StashActor`) in the level, set the static mesh on its `Marker` component (the chest's mesh; its collision stays BlockAll so it stops a walk, cuts the navmesh and answers the cursor), and move the `LabelWidget` component to sit above the chest if 140 uu up does not. `ReachRadius` (250 uu) is the only setting on the actor.

Tests: `SandboxARPG.Stash.*` in `Stash/Tests/StashTest.cpp`: the rules, the reach, gems, souls and fragments, item stacks (in, merge, a shard refused, out, the cap, a full stash, a full bag), the screen's order and press rule, the record's round trip with every kind, and a version-5 account loading empty.

## Dev commands

Compiled out of Shipping.

- `Slime.Stash.Deposit gem <bag index> | soul <DA_Soul_X> | frag <DA_Soul_X> | item <Needle|Lattice|Wheel> [PawnName]` and `Slime.Stash.Withdraw` with the same words (a gem by its stash index): the moves without the screen and without the chest's reach, as the bench's dev commands skip the bench's. On a joiner they forward themselves to the host for the joiner's own pawn.
- `Slime.Stash.Print [PawnName]`: the stash and the bag beside it, to the log.
- `Slime.Stash.Screen open|close|show`: opens the stash screen at the chest in reach (then the walk-away rule holds) or with no chest (then it never closes for distance), closes it, or logs its cells.
- `Slime.Stash.Chest spawn|hover|click`: until a chest is placed in the map, spawns one (the engine's cube) 200 uu in front of the local pawn on this machine, moves the cursor onto it so the HUD's own hover pass lights it, or left clicks it so the pawn's own press walks in and opens the stash.
- `Slime.Stash.Click bag:N|stash:N [plain]`: one Ctrl and left click (or a plain one) on the bag's Nth grid cell or the stash's Nth cell, sent through Slate as a real click, so the cells' own handlers answer it. A script proves the screen's path with it, not only the component's.

## What is placeholder or open

- The 80 cells and the 250 uu reach (register row `stash size`).
- The close range is the bench's 400 uu, reused rather than set apart.
- The pick-up key beside a chest that stands within the bench's or the keeper's reach opens the bench or the keeper first, as it always has between those two.
- A stash row has no cap of its own: two needle stacks deposited one after the other make one row of their sum, and a row past the bag's stack cap only comes back once the bag's stack has room for all of it.
