# Kit move phases

**Status:** first pass, provisional (register row `soul webs`). What a move of a v2 soul does between the press and its hits: the wind-up and its hold, Poise, charges, the recast, the leap, the channel, the toggle, kit projectiles, the chain, the repeat and the contact aura. All of it is read from the move's resolved row ([Soul webs](soul-webs.md)), on the server, for a player's pawn and a field monster alike. Every number is a column of `DT_MoveStats` and a placeholder.

## What you see

Some moves no longer hit the instant you press. A wind-up holds the body on its first frame while a red shape on the ground shows where the hit will land; a held key stretches some of them for more damage. A channel loops its clip and hits again every tick. A toggle stays on until you press it again or run dry. A move with charges shows a count on its bar slot. After some moves a second press within a short window fires the move again. The bar slot shows the phase, the charges and whether a toggle is on.

## The phases

**Wind-up** (`WindUp`). Seconds between the press and the hit. The pawn stands for it. The clip starts `WindUp` less its own hit delay after the press, so the clip's contact frame is the hit. A flinch does not interrupt it; only a stun ends it, and the log says why it was lost. A second press of the same move during its own wind-up is dropped. `OnWindUpEnd` is told when it fires.

**Hold** (`MaxHold`, `HoldIncreasedPerSecond`). After the wind-up's own time, the wait stretches while the key stays down, up to `MaxHold` seconds; the swing gains `HoldIncreasedPerSecond` per second held, as a fraction. Letting the key go fires it. Only a player holds a key: a field monster's wind-up never stretches.

**Poise** (`PoiseShare`, `PoiseMaxHits`). Each hit taken between the press and the move's first hit, landing or launch adds `PoiseShare` of the user's Defense as flat damage to the swing, for at most `PoiseMaxHits` hits (0: no limit). A row with no `WindUp` of its own (a leap) still gathers while it flies. Each such hit is told as `OnHitTakenInWindUp`.

**Charges** (`Charges`). A row with more than 1 stores that many uses (255 at most); 0 or 1 is a plain cooldown. A use takes one charge. Charges come back one at a time, each after the row's `Cooldown`, on one clock. The `GrantCharge` op gives charges back, never above the maximum. The maximum follows the ranks when a node changes it.

**Recast** (`RecastWindow`, `RecastShare`). For `RecastWindow` seconds after a use, a second press fires the move again at `RecastShare` of its damage, with no cost, no new cooldown and no new buff. The cooldown runs from the first use. A dash's recast goes back toward where the first began. A recast of a move with a contact aura ends the aura. A recast pressed during another move is dropped. `OnRecast` is told, and its hits answer the `HitIsRecast` condition.

**Leap** (`bLeap`, `LandRadius`). A dash that hits nothing on the way and lands one melee area hit of the row's flat damage in a circle where it ends. For the charge the capsule passes through every pawn, friend or enemy; walls still stop it. `OnDashEnd` is told at the landing.

**Other dash columns.** `bDashThroughEnemies` lets a dash pass through pawns the same way. `DragCount`: the first so many enemies crossed are moved to where the dash ends when it ends (not carried along it); never a Boss or a player-controlled pawn. `DashBounces`: when the dash ends it goes again toward the nearest enemy it has not crossed, within its own distance, once per bounce.

**Channel** (`ChannelSeconds`, `ChannelTick`, `ChannelManaPerTick`, `ChannelMoveScale`). The hit repeats every `ChannelTick` seconds, the first one tick after it began. With `ChannelSeconds` above 0 it runs that long; with 0 it runs while the key is held until the mana for a tick runs out. A pawn no player controls holds no key, so its unbounded channel runs for the length of the move's clip. `ChannelMoveScale` scales the walk speed (0 roots, 1 walks freely). Each tick is told as `OnChannelTick` and has its own budget of triggered effects; the end as `OnChannelEnd` with the reason logged.

**Toggle** (`bToggle`, `ManaPerSecond`). On until pressed again or the mana runs out. While it is on, the row's `BuffModifiers` are held as one modifier set (whatever `BuffDuration` says), its contact aura runs and its `SelfDamagePerSecond` is paid. Its cooldown starts when it ends, not when it is turned on. `OnToggleOn` and `OnToggleOff` are told.

**Self damage** (`SelfDamagePerSecond`). That fraction of maximum health per second while a toggle or channel runs, less the user's `SelfDamageLess` line (capped at 90 %). It is a price: never the last point, never a hit taken. Told as `OnSelfDamage`.

**The price of a use.** `HealthCostFraction` (of maximum health) and `HealthCostCurrentFraction` (of current health) are paid on use, never the last point, and told as `OnHealthPaid`. With `bCostsHealthInsteadOfMana`, a row that names a health price pays that and no mana; a row that names none pays its mana cost in health one for one. `ManaToHealthRate` pays the mana the user lacks in health at that rate. `SelfStateSpend` with `DamagePerStackSpent` spends every stack of a self state the user holds and adds that much flat damage per stack to every hit of the use. `RequireSelfState` refuses the press unless the stacks are held. A pawn no player controls pays no mana at all.

## Deliveries

**Kit projectiles** (`ProjectileCount` above 0). A fan of `ProjectileCount` across `ProjectileSpreadDegrees`, centred on the aim; one flies straight. Every projectile of a volley carries the whole hit. `ProjectilePierce` passes through that many enemies before it stops; `bProjectileReturns` turns it back to its shooter after its last hit, hitting what is in its path home; `ProjectileHoming` curves it toward the nearest enemy within that many degrees of its path. A projectile passes pawns of its own side. A row with no `ProjectileSpeed` flies at the soul asset's speed. `bProjectileLobbed`: one lob lands at the aim after its flight and hits `ImpactRadius`.

**Chain** (`ChainHops`, `ChainRadius`, `ChainShare`). A landed hit jumps to further enemies within the radius, each for the share of the one before; a `ChainShare` of 0 reads as 1. The `ChainHit` op does the same from a node. A repeat does not chain again. A hop answers `HitIsChain`.

**Repeat** (the `Repeat` op: `Delay`, `Share`, `Count`). The move's hit lands again so many times, so far apart, at the share. A repeat is not a use: no cost, no cooldown, no `OnUse`. It answers `HitIsRepeat` and has its own budget.

**Contact aura** (`ContactRadius`, `ContactPeriod`). While the row's buff or toggle runs, an enemy inside the radius is touched, then again each `ContactPeriod` it stays. A touch is the row's hit when it has damage, else its ailment, state and control alone.

**On each landed hit** the row's own columns act (`UKitAbility::OnKitHitLanded`): `ApplyState`, the ailment (`AilmentName` with `AilmentShare` or `AilmentFlat`), `EntranceSeconds`, `HealFractionOfDamage`; and once per swing, round its first target, `PullRadius` and a melee move's `ImpactRadius` splash. `OnSwingEnd` is told once per swing with the count of targets hit.

## What replicates

Outcomes never ride on a phase (GDD 9.8). The soul component replicates, push model:

- `KitPhase`: the running wind-up or channel (kind, the move's row, the server time it began, its duration, a channel's walk scale, the aim).
- `KitRecast`: the open recast window, shown when no wind-up or channel runs.
- `KitToggles`: the rows of the toggles that are on.
- `MoveChargesLeft` and `MoveChargesMax`, by moveset index; the existing cooldown ends say when the next charge comes back.

Every machine that shows the pawn holds the first frame for a wind-up and loops the clip for a channel from `KitPhase`. The owning client scales its own walk speed for a channel so it moves as the server does. The bar, the telegraph and the ground marks read these through `KitSeams::Phases()`.

## The release RPC

A client sends intent only: a press, and a key let go. `USoulComponent::RequestReleaseMove` is called when a slot key is released (`Player/MonsterCharacter.cpp`) and sends `ServerReleaseMove` only for a move whose row has `MaxHold` or an unbounded channel. The server validates the class is not null (a failure is reported to the failure-kick subsystem), rate-limits it per pawn like the press, and checks the move against the equipped soul's moveset; a move the soul lacks ends nothing and is logged. Then a held wind-up fires and a held channel ends.

## For testers

`Slime.Phase.Show` prints the local pawn's phase, its toggles and its charges. `Slime.ReleaseMove <AbilityClass>` lets a key go without a keyboard. `Slime.Kit.ClientUse <AbilityClass>` presses from any machine. `Slime.Kit.Column <AbilityRow> <Column> <Value>` sets a column for a proof run (this process only).

Log lines: `KitPhase: <pawn> winds up <row> (1.00 s to the hit: ...)`, `holds <move> (up to 1.50 s)`, `ends the wind-up of <move> (held 0.80 s, 24% increased)`, `loses the wind-up of <move>: <why>`, `takes hit 2 in the wind-up of <move>`, `'s Poise adds 12.0 flat damage to <move> (...)`, `spends a charge of <move> (1 of 3 left, the next back in 6.0 s)`, `has a charge of <move> back (2 of 3)`, `may recast <row> for 2.0 s`, `recasts <row> at 50% of its damage`, `channels <move> (a tick every 0.50 s, ...)`, `channel tick 3 of <row>`, `ends the channel of <row> after 6 tick(s): <why>`, `turns <row> on (...)`, `turns <row> off: <why> (cooldown 10.0 s)`, `loses 4.0 health to its own <row>`, `pays 8.0 health in place of 8.0 mana`. Deliveries: `Leap: <pawn> lands <row> at ...`, `Projectile: <pawn> fires 3 of <row> across 30 degrees (...)`, `Projectile: <pawn> lobs <row> at ...`, `Chain: <pawn>'s <row> jumps to <target> (hop 1 of 3, ...)`, `Aura: <pawn>'s <row> aura begins (...)`, `Dash: <pawn> sets down 2 dragged enemy(ies) ...`, `Dash: <pawn> bounces on to <target>`.

## What is not built

- A repeat of a projectile volley or of a dash's crossing: nothing lands, and the log says so. A repeat is built for a swing and for a leap's landing.
- A bounce returning to an enemy already hit.
- Entrance on a plate (see [states and ailments](kit-states-and-ailments.md)).
- Nothing here has been seen on screen by the team.

## For engineers

The phases are members of `USoulComponent`, defined in `Source/SandboxARPG/Kits/KitPhases.cpp` (wind-up, hold, Poise, charges, the health price, recast, channel, toggle) and `Kits/KitDeliveries.cpp` (leap, drag, bounce, projectiles, chain, aura); `Souls/SoulComponent.h` declares them and `ActivateMove` hands a kit move to `BeginKitMove`. `Kits/KitPhases.h` holds the replicated `FKitPhaseState`, the server-only `FKitMoveRuntime`, and the pure rules `FKitPhaseRules` (`MaxCharges`, `AdvanceCharges`, `SpendCharge`, `IsRecastOpen`, `PoiseFlat`, `HoldBonus`, `WindUpLead`, `ChannelTicks`, `ChannelTicksAffordable`, `SwingFlat`, `FanYaw`, `ChainDamage`, `SelfDamage`). `Kits/KitAbility.cpp` is the one ability body. `Abilities/MoveProjectile.*` carries pierce, return and homing. No tick: one timer per running phase. Every hit goes through the wrapper ASC's `ApplyHit` with the move's row name on it, so the web's `OnHit` answers it like a swing's. Tests: `SandboxARPG.Kits.Phases.*` (seven). Provisional markers: register row `soul webs`.
