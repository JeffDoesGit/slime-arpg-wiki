# The HUD

*Built in roadmap item E-2.17 (pull request #41). A placeholder layout: no design rule sets what the HUD shows, so this is marked provisional.*

## What you see

Two orbs at the bottom of the screen, red for health and blue for mana, that fill and drain as your numbers change. Between them, a bar of six ability slots labelled Q, W, E, R, LMB and RMB, and above that a thin gold experience bar with your level number at its left (see [Progression](progression.md)). When you die the whole screen dims and shows "You died", the experience lost, and a countdown until you are back. Each slot shows the icon of the move assigned to it, drawn edge to edge under the key label, or the move's name when its soul has no icon for it, or nothing (since E-2.60). While a move is on cooldown the slot goes dark and counts down the seconds; while you cannot afford it the slot goes dark and says "mana".

## Using it

- **Click a slot** to open a list of the moves your equipped soul knows, each with its icon beside the name, and pick one. Pick the same move on another slot and it moves there.
- **Press the slot's key** to perform the move (see [Abilities](abilities.md)). Only Q, W, E and R are bound today; the two mouse-button slots are drawn but do nothing.
- **Unequip your soul** and every slot clears. Equip a soul and its first move lands in the first empty slot on its own (since E-2.44), so you can fight before touching the picker; the picker still moves it wherever you like.

Slot assignments are a menu choice on your machine. They are never sent to the server and never saved, so two players in the same game can arrange their bars differently and nothing needs to agree.

## How it works, in plain terms

The orbs read the same health and mana numbers the server replicates to you; they never show a value the server has not confirmed. The same goes for the cooldown countdown: the server sends when each move will be ready and the slot compares that with the server's clock. The bar holds a list of six slot definitions loaded from a config file: the label, the key, and nothing else. A move's icon is a texture on its row of the soul asset (`FSoulMove::Icon`), looked up when a slot changes, never per frame; the eight Creepy and Corpse icons were generated in the soul-orb style and set by hand on the soul assets. Adding a slot, renaming one or changing its key is a config edit.

## What is waiting on design

- **Everything about layout.** Slot count, keys, what an orb should look like. The team has asked for a design item covering the HUD and hotbar; until then this is a placeholder shell, registered as such.
- **Orb art.** The fill is a flat disc rising inside a clipped box, drawn in code; no glow, no easing, no texture.

## For engineers

`Source/SandboxARPG/UI/PlayerHud.*` (the widget; its tick feeds each slot `USoulComponent::GetMoveCooldownRemaining` and Mana against the row's `ManaCost`), `UI/ResourceOrb.*`, `UI/AbilitySlotWidget.*` (`SetAvailability`; the icon image under the texts, button padding zeroed so it fills the square), `UI/HudLayoutSettings.*` (config-driven slot list, `AttackInPlaceKey`), `UI/MonsterHUD.*` (holds per-player slot assignments, in memory). All UI is C++ UMG; no `WBP_` assets. Provisional register rows "HUD ability slots" and "D-2.6".
