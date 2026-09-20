# Inventory screen

*Built in roadmap items E-2.10 (pull request #25) and E-2.13 (pull request #27). Design rules: GDD 3.1, only a few accessory slots; GDD 6.4, layout in data.*

## What you see

Press `I`. The screen has three parts:

1. **Three soul slots** at the top, primary, secondary and worn. The worn slot takes only a cosmetic soul (the human); dragging it there changes your body and nothing else, dragging it back takes it off.
   Primary and secondary: Drag a soul from the bag onto the primary slot to become that monster; drag it back to the bag to take it off. The secondary slot shows a name but refuses drops for now.
2. **A row of nine gem slots**: two red, two green, two blue, two that take any colour, one for special gems. Drag a [gem](gems.md) from the bag onto a slot to equip it; the slot shows the gem's stone (a ruby for red, an emerald for green, a sapphire for blue, an onyx for the special class) and, on hover, its lines. Drag it back to the bag to take it off. A gem dropped on a slot of the wrong colour stays in the bag; the server refuses it.
3. **A bag grid**, 4 rows by 10, showing the souls you own with their icons, then the souls still being assembled as dimmed icons with their fragment count (E-2.7), then your gems as their stones with a tooltip of their lines (since E-2.61; a colour with no icon configured falls back to a flat colour swatch).

That is the whole screen. No stash yet (it needs a city), no rarity display, no sorting.

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

`Source/SandboxARPG/UI/MonsterHUD.*` (owns the screens), `UI/InventoryScreen.*`, `UI/InventoryLayoutSettings.*` (the settings class), `UI/SoulSlotWidget.*`, `UI/SoulItemCell.*`, `UI/SoulDragDropOperation.h`, `UI/SoulEquipButton.*`; for gems (E-3.5) `UI/GemSlotWidget.*`, `UI/GemItemCell.*`, `UI/GemDragDropOperation.h`, refreshed from `UGemEquipmentComponent::OnGemsChanged`; `UGemItemCell::PaintSwatch` paints a cell, a slot or the drag visual with the colour's icon from `UInventoryLayoutSettings::GemIcons` (`DefaultGame.ini`), loaded on refresh, or the flat tint when the colour has no row. The bag grid is re-laid on every refresh from two cell pools: soul cells for owned and partly assembled souls, then gem cells; a cell is a gem cell only when its gem index is one the bag holds (E-2.58 fixed the fragment cell that used to fall into the gem branch with a negative index). Config in `DefaultGame.ini` clears the settings array first and appends with `.` rather than `+`, because `+` drops entries identical to one already present and collapsed the colour pairs.
