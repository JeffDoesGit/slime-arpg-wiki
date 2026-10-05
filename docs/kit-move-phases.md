# Kit move phases

**Status:** first pass, provisional (register row `soul webs`). What a move of a v2 soul does between the press and its hits: the wind-up and its hold, Poise, charges, the recast, the leap, the channel, the toggle, kit projectiles, the chain, the repeat and the contact aura. All of it is read from the move's resolved row ([Soul webs](soul-webs.md)), on the server, for a player's pawn and a field monster alike. Every number is a column of `DT_MoveStats` and a placeholder.

## What you see

Some moves no longer hit the instant you press. A wind-up holds the body on its first frame while a red shape on the ground shows where the hit will land; a held key stretches some of them for more damage. A channel loops its clip and hits again every tick. A toggle stays on until you press it again or run dry. A move with charges shows a count on its bar slot. After some moves a second press within a short window fires the move again. The bar slot shows the phase, the charges and whether a toggle is on.

## The phases

**Wind-up** (`WindUp`). Seconds between the press and the hit. The pawn stands for it. The clip starts `WindUp` less its own hit delay after the press, so the clip's contact frame is the hit. A flinch does not interrupt it; only a stun ends it, and the log says why it was lost. A second press of the same move during its own wind-up is dropped. `OnWindUpEnd` is told when it fires.

**Hold** (`MaxHold`, `HoldIncreasedPerSecond`). After the wind-up's own time, the wait stretches while the key stays down, up to `MaxHold` seconds; the swing gains `HoldIncreasedPerSecond` per second held, as a fraction. Letting the key go fires it. Only a player holds a key: a field monster's wind-up never stretches.

**Poise** (`PoiseShare`, `PoiseMaxHits`). Each hit taken between the press and the move's first hit, landing or launch adds `PoiseShare` of the user's Defense as flat damage to the swing, for at most `PoiseMaxHits` hits (0: no limit). A row with no `WindUp` of its own (a leap) still gathers while it flies. Each such hit is told as `OnHitTakenInWindUp`.

**Charges** (`Charges`). A row with more than 1 stores that many uses (255 at most); 0 or 1 is a plain cooldown. A use takes one charge. Charges come back one at a time, each after the row's `Cooldown`, on one clock. The `GrantCharge` op gives charges back, never above the maximum. The maximum follows the ranks when a node changes it.

**Recast** (`RecastWindow`, `RecastShare`). For `RecastWindow` seconds after a use, a second press fires the move again at `RecastShare` of its damage, with no cost, no new cooldown and no new buff. The cooldown runs from the first use. A dash's recast goes back toward where the first began unless the row's `RecastMove` says otherwise: `ToAim` runs the dash or leap again toward the recast's own aim (a second landing), `None` moves nothing and lands none of the row's dash hits, so the `OnRecast` ops are the recast. A recast of a move with a contact aura ends the aura. A recast pressed during another move is dropped. `OnRecast` is told, and its hits answer the `HitIsRecast` condition.

**Leap** (`bLeap`, `LandRadius`). A dash that hits nothing on the way and lands one melee area hit of the row's flat damage in a circle where it ends. For the charge the capsule passes through every pawn, friend or enemy; walls still stop it. `OnDashEnd` is told at the landing. A leap lands at the aim when the aim is nearer than the row's `DashDistance` and flies that distance when the aim is further (`FKitPhaseRules::LeapDistance`); a plain dash always runs its whole distance, and a recast leap still goes back to where the first began.

**Other dash columns.** `bDashThroughEnemies` lets a dash pass through pawns the same way. `DragCount`: the first so many enemies crossed are moved to where the dash ends when it ends (not carried along it); never a Boss or a player-controlled pawn. `DashBounces`: when the dash ends it goes again toward the nearest enemy it has not crossed, within its own distance, once per bounce.

**Channel** (`ChannelSeconds`, `ChannelTick`, `ChannelManaPerTick`, `ChannelMoveScale`). The hit repeats every `ChannelTick` seconds over the move's rate (attack or cast speed, a slow, a node's or a rule's `PlayRate`), the first one tick after it began; every tick reads the row again and arms the next, so a rate that changes while the channel runs is the next tick's, and a tick's extra hit delays shrink with it. A press of another move ends the channel, as letting its key go would, and fires in the same frame; a row with `bUsableWhileBusy` fires without ending it. With `ChannelSeconds` above 0 it runs that long; with 0 it runs while the key is held until the mana for a tick runs out. A pawn no player controls holds no key, so its unbounded channel runs for the length of the move's clip. `ChannelMoveScale` scales the walk speed (0 roots, 1 walks freely). Each tick is told as `OnChannelTick` and has its own budget of triggered effects; the end as `OnChannelEnd` with the reason logged.

**Toggle** (`bToggle`, `ManaPerSecond`). On until pressed again or the mana runs out. While it is on, the row's `BuffModifiers` are held as one modifier set (whatever `BuffDuration` says), its contact aura runs and its `SelfDamagePerSecond` is paid. Its cooldown starts when it ends, not when it is turned on. `OnToggleOn` and `OnToggleOff` are told.

**Self damage** (`SelfDamagePerSecond`). That fraction of maximum health per second while a toggle or channel runs, less the user's `SelfDamageLess` line (capped at 90 %). It is a price: never the last point, never a hit taken. Told as `OnSelfDamage`.

**The price of a use.** `HealthCostFraction` (of maximum health) and `HealthCostCurrentFraction` (of current health) are paid on use, never the last point, and told as `OnHealthPaid`. With `bCostsHealthInsteadOfMana`, a row that names a health price pays that and no mana; a row that names none pays its mana cost in health one for one. `ManaToHealthRate` pays the mana the user lacks in health at that rate. `SelfStateSpend` with `DamagePerStackSpent` spends every stack of a self state the user holds and adds that much flat damage per stack to every hit of the use. `RequireSelfState` refuses the press unless the stacks are held. A pawn no player controls pays no mana at all.

## Deliveries

**Kit projectiles** (`ProjectileCount` above 0). A fan of `ProjectileCount` across `ProjectileSpreadDegrees`, centred on the aim; one flies straight. Every projectile of a volley carries the whole hit. `ProjectilePierce` passes through that many enemies before it stops; `bProjectileReturns` turns it back to its shooter after its last hit, hitting what is in its path home; `ProjectileHoming` curves it toward the nearest enemy within that many degrees of its path. A projectile passes pawns of its own side. A row with no `ProjectileSpeed` flies at the soul asset's speed. `bProjectileLobbed`: one lob lands at the aim after its flight and hits `ImpactRadius`.

**A projectile's memory.** Each kit projectile keeps what it has done: how far it has flown (both legs of a returning one), how many enemies it hit before, whether it is on its way back, whether the straight line of its path crossed one of its shooter's placed things, its place in the cast and whether a node added it past the table row's count, and (shared by the cast) how many other projectiles of the cast landed on each target. While its hit resolves the shooter's web can ask through the `Projectile*` conditions and the per-pierce and per-distance amounts (schema 7.16); the server log says them on every hit (`AMoveProjectile: ... hits ...: flown 764 uu, hit index 0, ...`). A lob has no actor and no memory.

**Chain** (`ChainHops`, `ChainRadius`, `ChainShare`). A landed hit jumps to further enemies within the radius, each for the share of the one before; a `ChainShare` of 0 reads as 1. The `ChainHit` op does the same from a node. A repeat does not chain again. A hop answers `HitIsChain`.

**Repeat** (the `Repeat` op: `Delay`, `Share`, `Count`). The move's hit lands again so many times, so far apart, at the share. A repeat is not a use: no cost, no cooldown, no `OnUse`. It answers `HitIsRepeat` and has its own budget.

**Contact aura** (`ContactRadius`, `ContactPeriod`). While the row's buff or toggle runs, an enemy inside the radius is touched, then again each `ContactPeriod` it stays. A touch is the row's hit when it has damage, else its ailment, state and control alone.

**On each landed hit** the row's own columns act (`UKitAbility::OnKitHitLanded`): `ApplyState`, the ailment (`AilmentName` with `AilmentShare` or `AilmentFlat`), `EntranceSeconds`, `HealFractionOfDamage`; and once per swing, round its first target, `PullRadius` and a melee move's `ImpactRadius` splash. A swing that reaches nobody still pulls, around the centre of its shape (`USoulComponent::ResolveMeleeHit`). A row with no damage that names a shape swings as a touch, and a kit dash with no damage crosses as one: `OnHit` and these columns run, nobody is hurt ([soul webs](soul-webs.md), the op dispatcher). `OnSwingEnd` is told once per swing with the count of targets hit.

## What the fix pass of 2026-10-04 changed

- A wind-up no longer than the clip's own hit delay still ends (on the step after the press) and tells `OnWindUpEnd`.
- A charge spent from full starts its clock at the spend, whatever an older plain cooldown left behind.
- A recast whose row says `RecastShare` 0 makes no hit. A recast pressed inside the busy window is dropped and said in the log once a second. A row with `bCooldownAfterRecast` starts its cooldown over when the recast is used or its window lapses.
- A Repeat lands where its swing came down, for its share of the largest hit that swing landed; it carries no control, no pull and no row rider, and takes only the standing rules that ask `HitIsRepeat`.
- `Dash` and `Leap` are stamped when the dash begins, so `WithinSecondsOf` holds for the hits the dash itself makes; the press that turns a toggle off stamps `Use`.
- A leap's row pull lands before the landing's hit, and who it brings inside the landing is hit.
- A chain hop takes the first hit's roll (no miss and no crit of its own), and a kit projectile's landed hit chains by its row.

## What the timing pass of 2026-10-04 changed

- **The swing follows the rate, the clip is clamped.** A move's rate is its resolved `PlayRate` (attack or cast speed and node overrides on it, less a slow) and has no ceiling. Hit delays, the time before the same move may fire again and the commit window follow it; only the clip on the mesh is held between the `MoveClipRateMin` and `MoveClipRateMax` rows of `DT_CombatRules` (0.1 and 4). A clip that outlasts its swing is taken over by the next press. The log says both: `swing of 'KitOniBasic': 0.42 s at x6.25, first hit 0.21 s, commit 0.34 s (clip x4.00)`.
- **A derived commit window shrinks with the rate.** The generator writes `CommitRate` beside a `CommitSeconds` it derives; the window is then `CommitSeconds x CommitRate` over the rate now. A window a sidecar wrote, and a dash's, stay real seconds.
- **A press during a channel.** Another move ends the channel and fires (`ends the channel of <row> after N tick(s): another move was pressed`); a `bUsableWhileBusy` row fires and the channel ticks on; nothing is pressed through a wind-up. The owning client sends such a press instead of dropping it (`USoulComponent::GetKitBusyPress`).
- **A channel's tick is armed again every tick** at the rate the row resolves to then.
- **`RecastMove`** on a dash or leap row: `Back`, `ToAim`, `None`.
- **Equipping a soul asks for what its moves play** (clips, effects, kit cues) in one asynchronous request held while the soul is worn, so the first press of a move loads nothing inside the press.

## What replicates

Outcomes never ride on a phase (GDD 9.8). The soul component replicates, push model:

- `KitPhase`: the running wind-up or channel (kind, the move's row, the server time it began, its duration, a channel's walk scale, the aim).
- `KitRecast`: the open recast window, shown when no wind-up or channel runs.
- `KitToggles`: the rows of the toggles that are on.
- `MoveChargesLeft` and `MoveChargesMax`, by moveset index; the existing cooldown ends say when the next charge comes back.

Every machine that shows the pawn holds the first frame for a wind-up and loops the clip for a channel from `KitPhase`. The owning client hands the channel's scale to its own pawn's walk speed (`AMonsterCharacter::SetKitWalkScale`), one of the three inputs it shares with the authority (the MoveSpeed attribute, a slow, the channel), so it moves as the server does through a stance or a slow that begins or ends during the channel. The bar, the telegraph and the ground marks read these through `KitSeams::Phases()`.

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

## The ops pass of 2026-10-04

- **`bOneCritRoll`** (move column). A swing or a leap's landing of the row rolls its crit on the first enemy it lands on; the others take that answer (`FHitSpec::bCritDecided`), each multiplying its own hit. Each hit index of a multi-hit swing is its own roll.
- **A forced crit** (`ForceCrit` with a `Count`) is spent on a swing: every enemy the same ability, use and hit index lands on crits.
- **`SelfStateSpend`**: the stacks a use spent at its press are remembered for that use (`USoulComponent::GetKitUseSpent`), so an `OverrideRules` rule with `AddPerStack` of that state still reads them when the row is resolved for the delivery.
- **The press is pending** from the cost check until the web counts the use: `UseIndexEvery` in an `OverrideRules` rule on a cost column reads the use being pressed.
- **The dash distance** (`LastDashDistanceAbove`) is taken when the dash has run, before its landing hit; a recast and a bounce measure from where they began. **The channel tick's number** is known to the tick's own swing.
- **A drag** tells `OnControlApplied` (`Pull`) for each enemy set down.
- **`SelfDamagePerSecond`** is paid by a player's pawn only.
