# Inventory screen

*Built in roadmap items E-2.10 (pull request #25) and E-2.13 (pull request #27). Design rules: GDD 3.1, only a few accessory slots; GDD 6.4, layout in data.*

## What you see

Press `I`. The screen has three parts:

1. **Three soul slots** at the top, primary, secondary and worn. The worn slot takes only a cosmetic soul (the human); dragging it there changes your body and nothing else, dragging it back takes it off.
   Primary and secondary: Drag a soul from the bag onto the primary slot to become that monster; drag it back to the bag to take it off. The secondary slot shows a name but refuses drops for now.
2. **A row of nine gem slots**: two red, two green, two blue, two iridescent (only an iridescent gem, since E-3.6; they took any colour before), one for special gems. Drag a [gem](gems.md) from the bag onto a slot to equip it; the slot shows the gem's stone (a ruby for red, an emerald for green, a sapphire for blue, an onyx for the special class) and, on hover, its lines. Drag it back to the bag to take it off. A gem dropped on a slot of the wrong colour stays in the bag; the server refuses it.
3. **A bag grid**, 4 rows by 10, showing the souls you own with their icons, one cell a soul with its copies counted (E-2.251), then the souls still being assembled as dimmed icons with their fragment count (E-2.7), then your gems as their stones with a tooltip of their lines (since E-2.61; a colour with no icon configured falls back to a flat colour swatch). Since E-3.8 a gem cell's frame, and an equipped gem's slot frame, is the colour of the gem's rarity, and the tooltip shows each line's value in its own format with the line's tier and roll band beside it (see [Gems](gems.md)).

That is the whole screen. No stash yet (it needs a city), no sorting.

**The look since E-2.180 (2026-10-06).** The screen is one stone panel with a gold frame docked on the right edge of the view, 620 by 880 at 1080 lines, after the mockup's board 2: the three souls as discs with gold rings (the primary larger, its ring brighter; "WORN", "SOUL", "SECONDARY" beneath), the nine sockets as sunk wells rimmed in their colour (an empty socket shows only its rim and names itself on hover), the bag on a sunk well with one-pixel lines between its cells, a hint and the close button along the bottom. A gem's tooltip is a dark card in Diablo's manner: the stone's name and its rarity in the rarity's colour, one line per affix in blue with its tier, and "Socketed: Red" beneath an equipped one. Drag, right-click, Shift-right-click and every refusal are the same handlers as before; the theme only dresses the widgets. The old look and its switch were deleted by E-2.193. Provisional (register row `ui look`). Dev: `Slime.ToggleInventory` opens it with no keyboard, for a frame of a game rendered off screen.

## Where the layout comes from

The gem slot list, their colours, the icon per gem colour and the bag size are project settings (Project Settings › Game › Slime ARPG Inventory Layout), so a designer can change them without touching code. The nine-slot row is transcribed from the team's Miro board and marked provisional: the number of slots and what each colour means are still design questions (D-3.2). The bag size has no design rule at all and is a placeholder.

## How it is built

Every widget on the screen is built in code, not in the Unreal widget designer. That is deliberate for the prototype: it keeps the screen out of binary asset files that cannot be merged. A designer-made widget can replace it later without changing how it talks to the game.

Dragging a soul never changes anything by itself. The drop sends a request to the server, the server decides, and the screen redraws from the answer.

A right-click on a bag cell sends the same request its drag would (E-2.86): a soul goes to the primary slot, a cosmetic soul to the worn slot, a gem to the first empty slot that takes its colour, coloured slots before the two any-colour ones so those are not spent on a gem that has its own colour. With no fitting slot free the click logs `no free slot for a <colour> gem` and nothing is sent. The server checks a right-click exactly as it checks a drop; the cell only picks the target. Right-click on a slot does nothing; drag a slotted item back to the bag to take it off.

## What is waiting on design

- **Gem slots and colours** (D-3.2): count is provisional, meaning open.
- **Accessory slots** and the items that go in them (Phase 3).
- **Bag size** and **the stash** in cities.
- **Fragment counts** next to each soul, once soul drops exist.

## For engineers

`Source/SandboxARPG/UI/MonsterHUD.*` (owns the screens), `UI/HudScreen.*` (the screens' base since E-2.92: panel clicks stay on the screen, the X and Escape close it; see [the HUD](hud.md)), `UI/InventoryScreen.*`, `UI/InventoryLayoutSettings.*` (the settings class), `UI/SoulSlotWidget.*`, `UI/SoulItemCell.*`, `UI/SoulDragDropOperation.h`, `UI/SoulEquipButton.*`; for gems (E-3.5) `UI/GemSlotWidget.*`, `UI/GemItemCell.*`, `UI/GemDragDropOperation.h`, refreshed from `UGemEquipmentComponent::OnGemsChanged`; `UGemItemCell::PaintSwatch` paints a cell, a slot or the drag visual with the colour's icon from `UInventoryLayoutSettings::GemIcons` (`DefaultGame.ini`), loaded on refresh, or the flat tint when the colour has no row. The bag grid is re-laid on every refresh from two cell pools: soul cells for owned and partly assembled souls, then gem cells; a cell is a gem cell only when its gem index is one the bag holds (E-2.58 fixed the fragment cell that used to fall into the gem branch with a negative index). Config in `DefaultGame.ini` clears the settings array first and appends with `.` rather than `+`, because `+` drops entries identical to one already present and collapsed the colour pairs.

## Jon's first look fixes (E-2.190)

*Built 2026-10-06.* In the themed bag a gem's cell is a dark well with a one-pixel line in the gem's rarity colour; before, the line's colour filled the whole cell behind the stone (white for a common gem). The themed soul slots take drags again: a soul dragged from the bag lands on the primary, secondary or worn slot (the worn slot still takes only a cosmetic soul), a slot's orb dragged into the bag unequips it, and the primary's orb dragged onto the secondary slot, or the other way, swaps the two souls; a right-click still equips the primary and a Shift-right-click the secondary. A soul in the secondary slot now leaves the bag, as the primary's does. The X key (rebindable, "Swap souls" on the options page) swaps the primary and secondary souls in the field: the body, moves and stats follow the primary as on any equip, the server decides, and `Slime.SoulSwapCooldown` is the wait between swaps (0 today). Provisional on the D-2.17 memo (register row D-2.17). For engineers: `USoulSlotWidget::BuildThemed` sets the widget `Visible` so Slate routes presses and drops to it; `USoulComponent::SwapSouls`, `RequestSwapSouls`, `ServerRequestSwapSouls`; `Slime.SwapSouls` and `Slime.DragTo` for a keyboard-free check.

## Souls in the bag, fragments and a full bag (E-2.241, E-2.249, E-2.251, 2026-10-21)

- **One cell a soul.** Every soul you own and are not wearing has one cell, whatever its copies: a "x2" in the corner counts the copies from two, and the tooltip says how many fragments you hold toward the next ("Golem x2 (1 of 3 fragments toward another copy)"). A soul you are still collecting shows as the dimmed cell with its count ("1/3"), as before. Souls cannot be dropped on the ground or lost: a soul is an unlock (GDD 2.8). Found at the playtest of 2026-10-20: a joiner who farmed Golems and Vetalas had nine copies and dead "0/3" cells crowding the bag, none of which could leave it.
- **Souls never fill the bag.** The forty cells are the gems'. Soul cells sit above them and take nothing from them, so a soul, a fragment or a copy always comes in; a gem you walk over with forty gems in the bag stays on the ground and the notice line reads "Inventory full". The refusal of souls and fragments by a full bag that E-2.249 added is gone with this.
- **Fragments read right.** A soul being assembled shows its count until the third fragment; then the soul appears, and the dimmed "(0/3)" cell that used to stay behind is gone.
- **A second copy of a soul stays in the bag.** Taking a soul off, swapping or moving it back could merge two copies into one; each copy now keeps its own cell.

## Dropping fragments (E-2.255, 2026-10-21)

*Jon, from his Standalone look at the live build of 2026-10-21: "player should be able to drop fragments on the ground if they want to".* A soul you are still collecting (the dimmed "1/3" cell) can be dragged out of the bag onto the play view: every fragment you hold of that soul lands as one pickup a step in front of you, labelled "Vetala fragments x2", and the cell is gone. Anyone who walks onto the pickup takes the fragments, which count toward their next copy of that soul as a kill's would, assembling it on the way if they complete the row; the pickup never returns and is gone after 180 seconds untaken (the hand-dropped gem's time). Whole souls still never leave the bag (GDD 2.8), and a fragment pickup cannot be dropped onto a soul slot. The drop is the whole row, not one fragment at a time.

For engineers: `USoulItemCell::NativeOnDragCancelled` (a fragment cell's drag names no soul and carries `USoulDragDropOperation::FragmentSoul`), `USoulComponent::RequestDropFragments` / `ServerRequestDropFragments` (validated, the `SoulEquip` rate row) / `DropFragments` (authority: the row removed, `ASoulPickup` spawned deferred and given its count by `InitFragments` with the monster row's fragments-per-soul and `PlayerDropLifetimeSeconds`), `ASoulPickup::TryGiveTo` (fragments through `AddFragment` at that count). `USoulComponent::FragmentsPerSoulFor` is the one lookup of the monster row's number. Dev: `Slime.AddFragments <DA_Soul_X> [Count]`, `Slime.DropFragments <DA_Soul_X>`. Provisional on Jon's words against the D-2.1 memo (fragments as a count on the character, not an item); no written recommendation exists for dropping them.
