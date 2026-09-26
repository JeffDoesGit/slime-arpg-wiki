# Flasks

**Status:** first pass, provisional on D-2.11 (built 2026-09-26 from the Research Bank's potion and flask study; Jon: "implement recommendation in one pass, I will iterate later"). Every number is a placeholder row in `DT_CharacterRules`.

## What you see

Two flask pictures on the ability bar, the red health flask left of slot 1 and the blue mana flask right of the RMB slot, each larger than the round Skills, Options and Arena buttons, with its key under it. The picture is the gauge: full, two thirds, one third or empty, one drink per third. Press the key or click the flask and it pours: your orb climbs over the next few seconds, the picture dims and shows the seconds left; when a drink cannot be paid for the flask reddens. Kill things and the counts go up. Walk into a town or bind a checkpoint well and both counts fill.

## The rules

- **A flask is not an item.** You never pick one up, buy one, or carry one; there is no slot and no affix line for it. Both flasks are part of your character.
- **Charges.** Each flask holds 60 charges and a drink costs 20, so a full flask is three drinks. Every kill near you (within 2500 units of the monster, whoever landed the blow) adds charges to both flasks by the monster's tier: 5 for a normal monster, 10 for an elite, 20 for a boss. Nothing comes back on its own.
- **The drink.** The health flask restores 40 % of your maximum health over 3 seconds; the mana flask 30 % of your maximum mana over 2 seconds. The pour stops the moment you are full, so a drink at high health is a waste of charges. A second press while the first is still pouring is refused, as is a press with fewer than 20 charges or while dead.
- **Refills.** Binding a checkpoint well (the well at each zone entrance, which also fills your health and mana the first time) and entering a safe zone such as the town fill both flasks to 60. Nothing else does.
- **Held in reserve.** A trickle row (`FlaskTrickleSeconds`, 0 and off) can give one drink's worth of charges back every N seconds if playtests show slow builds running dry; that is Diablo IV's Season 11 rule and it is not on.

## What it is not, yet

The keys are rebindable on the options screen (Health flask, Mana flask). The four pictures per flask were generated on 2026-09-26 (`Art Assets/UI/Icons/Flasks/T_Icon_Flask_<Health|Mana>_<0..3>.png`, the lower states edited from the full one so the bottle stays the same); there is no pour animation. There is no server rule to scale or switch flasks off, no flask affix, no potion drop, no upgrade. Whether the mana flask stays at all, whether kills credit everyone nearby or only the killer, and whether anything besides a well refills are the questions the research leaves for Duilio (D-2.11).

## For developers

`Source/SandboxARPG/Player/FlaskComponent.*` on the player pawn: replicated charges and recovery end times (push model, C-3), a 0.1 s recovery timer per flask on the authority that adds through `ApplyModToAttribute` and stops at the maximum, `RequestUse` (local on the authority, `ServerRequestUseFlask` with validation and the rate table row otherwise), `CreditKillNearby` called from `AFieldMonster::HandleHealthDepleted` with the row's `Tier`, `RefillAll` from `AMonsterCharacter::BindCheckpoint` and `SetInSafeZone(true)`. The pure rules (`EvaluateKillCharges`, `EvaluateSpend`, `EvaluateTickAmount`) are tested in `Player/Tests/FlaskRulesTest.cpp`. Keys: `HealthFlaskKey` and `ManaFlaskKey` in `UHudLayoutSettings` (`DefaultGame.ini`, Q and E), rebindable as `UPlayerOptions::ActionHealthFlask` and `ActionManaFlask`. The HUD buttons are `UPlayerHud::BuildFlask` (a size box of `FlaskButtonSize`, a drawless button, an overlay of the picture, the key and the seconds) and `RefreshFlask` each frame without allocation; the pictures are `HealthFlaskIcons` and `ManaFlaskIcons` in `UHudLayoutSettings` (index = drinks left), loaded once at build, a tinted disc when fewer than four are set. Dev commands: `Slime.Flask health|mana`, `Slime.ShowFlasks`, `Slime.SetFlaskCharges <n>`.
