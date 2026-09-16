# The HUD

*Built in roadmap item E-2.17 (pull request #41). A placeholder layout: no design rule sets what the HUD shows, so this is marked provisional.*

## What you see

Two orbs at the bottom of the screen, red for health and blue for mana, that fill and drain as your numbers change. Between them, a bar of six ability slots labelled 1, 2, 3, 4, LMB and RMB, and above that a thin gold experience bar with your level number at its left (see [Progression](progression.md)). When you die the whole screen dims and shows "You died", the experience lost, and a countdown until you are back. Each slot shows the icon of the move assigned to it, drawn edge to edge under the key label, or the move's name when its soul has no icon for it, or nothing (since E-2.60). While a move is on cooldown the slot goes dark and counts down the seconds; while you cannot afford it the slot goes dark and says "mana". Over every other character stands a small plate with its name and a health bar (since E-2.70): green for someone your character would not hurt, another player, and red for something it would, a field monster. Your own character has no plate; the orbs are yours. A dead character's plate goes away with it.

## Using it

- **Click a slot** to open a list of the moves your equipped soul knows, each with its icon beside the name, and pick one. Pick the same move on another slot and it moves there.
- **Press the slot's key** to perform the move (see [Abilities](abilities.md)). Only 1, 2, 3 and 4 are bound today; the two mouse-button slots are drawn but do nothing.
- **Unequip your soul** and every slot clears. Equip a soul and its first move lands in the first empty slot on its own (since E-2.44), so you can fight before touching the picker; the picker still moves it wherever you like.

Slot assignments are a menu choice on your machine. They are never sent to the server and never saved, so two players in the same game can arrange their bars differently and nothing needs to agree.

## How it works, in plain terms

The orbs read the same health and mana numbers the server replicates to you; they never show a value the server has not confirmed. The same goes for the cooldown countdown: the server sends when each move will be ready and the slot compares that with the server's clock. The bar holds a list of six slot definitions loaded from a config file: the label, the key, and nothing else. A move's icon is a texture on its row of the soul asset (`FSoulMove::Icon`), looked up when a slot changes, never per frame; the eight Creepy and Corpse icons were generated in the soul-orb style and set by hand on the soul assets. Adding a slot, renaming one or changing its key is a config edit. A name plate is a widget the pawn itself carries, drawn on the screen above its capsule rather than in the world, so a wall never hides it and distance never shrinks it; it reads the same replicated health as the orbs, and asks your character whether it is hostile to that pawn to pick its colour, which is why a PvP rule would turn other players red without a UI change. The name is the player's name from the session, or for a monster the display name of the soul it wears.

## The debug menu (F1)

*Roadmap item E-2.73. Dev tooling, not a game feature: it exists in editor and Development builds and is compiled out of Shipping the same way the `Slime.*` console commands are.*

Press **F1** and a panel opens at the left edge of the screen; press it again and it closes. It is hidden until then. At the top it shows your character's name, whether this machine is the host or a client, your level, the zone you stand in, health and mana, and whether you are dead or in a safe zone. Below that, a Souls block: a search box, a dropdown of every soul asset in the project (a new soul shows up by itself) and Grant, Equip primary and Wear buttons for the one picked. Then buttons: set a level or add experience; drop health or mana to zero or fill them; grant a gem of each colour; spawn a number of Corpse or Creepy monsters in front of you; switch the movement-rooting and attack-feel toggles; two camera presets; and buttons that print attributes, gems and the server rules to the Output Log. A **target pawn** dropdown lists every character in the world so a button can act on another player's character instead of your own. A text box at the bottom runs any console command you type, and the last line the command logged is echoed under it.

Every button runs one existing console command on your machine, exactly as if you had typed it. That means the same rule applies as at the console: things that change the game (granting, levelling, spawning, setting health) only work on the host, because the server owns those outcomes; on a joiner's machine the button logs "run on the host" and nothing happens. The feel toggles and the camera are local and work anywhere. Nothing new is sent to the server and nothing is saved.

## What is waiting on design

- **Everything about layout.** Slot count, keys, what an orb should look like. The team has asked for a design item covering the HUD and hotbar; until then this is a placeholder shell, registered as such.
- **Orb art.** The fill is a flat disc rising inside a clipped box, drawn in code; no glow, no easing, no texture.

## For engineers

`Source/SandboxARPG/UI/PlayerHud.*` (the widget; its tick feeds each slot `USoulComponent::GetMoveCooldownRemaining` and Mana against the row's `ManaCost`), `UI/ResourceOrb.*`, `UI/AbilitySlotWidget.*` (`SetAvailability`; the icon image under the texts, button padding zeroed so it fills the square), `UI/HudLayoutSettings.*` (config-driven slot list, `AttackInPlaceKey`), `UI/MonsterHUD.*` (holds per-player slot assignments, in memory; `ToggleDebugMenu` under `#if !UE_BUILD_SHIPPING`), `UI/DebugMenuScreen.*` (E-2.73: `UDebugMenuButton` carries a command template, `{pawn}` `{level}` `{xp}` `{count}` filled at the click, run through `APlayerController::ConsoleCommand` with an `FOutputDevice` on `GLog` for the echo line; the header is polled in `NativeTick` and the text rebuilt only when a whole number changes; `DebugMenuKey` in `UHudLayoutSettings`, bound in `AMonsterCharacter::SetupPlayerInputComponent` outside Shipping), `UI/NamePlateWidget.*` (the plate; `Setup(pawn)`, `RefreshName`, a tick that polls Health and re-picks the colour only when the viewer's pawn changes) hung from `AMonsterCharacter::NamePlate`, a screen-space `UWidgetComponent` re-placed by `RefreshNamePlate` when the capsule, the player state or the death flag changes; sizes and colours in `UHudLayoutSettings` ("Name Plates"). All UI is C++ UMG; no `WBP_` assets. Provisional register rows "HUD ability slots" and "D-2.6".
