# Kit states and ailments

**Status:** first pass, provisional (register row `soul webs`). The counted states, meters and damaging ailments the [soul webs](soul-webs.md) read and write, the row of discs that shows them, and the caps page. Every number is a row's placeholder. The older Splinter counts and the legacy poison stacks ([Level and growth](progression.md)) stay as they are for the legacy kits and are not this system.

## What you see

Small discs under a pawn's name plate and, larger, above your own ability bar. Each disc is one state or one ailment: the row's colour, its letter, the stack or instance count beside it, and a ring that drains with its time. A meter on your own row is a thin bar with its amount. A plate shows the states others may see on that pawn and every ailment on it; your own row shows the states only you see (the self states) and your ailments. There are no icons yet: the coloured disc and letter are placeholders.

## States

A state is a row of `DT_KitStates` (71 rows today: Sunder, Splinter, Static, Tempo, Morsel, Gristle, Tolls, Coil, Stone Time and the rest). It is either a **count** (whole stacks) or a **meter** (a float pool).

- **Whose pile.** A row with `bOnSelf` is the holder's own, whoever added to it (Tempo, Morsel). A row with `bShared` is one pile for everyone who applies it (Sunder, the one armour shred). Any other state is counted per applier: my Splinters on a monster are mine, yours are yours.
- **Cap.** `Cap` stacks or units; for a meter with `CapFractionOfMaxHealth` above 0, that fraction of the holder's maximum health instead. A gain never passes the cap. A meter that absorbs hits is also held to the `HealthPoolCap` rule row.
- **How it runs out** (`DecayMode`): `AllAtOnce` (every stack goes when `Life` ends), `PerStack` (each stack has its own `Life`; at the cap the oldest gives way), `OnePerInterval` (one stack every `Life` seconds), `Drain` (a meter: after `DecayDelay` without a gain, `DecayPerSecond` leaves each second, or that fraction of maximum health with `DecayFractionOfMaxHealth`), `Never`. With `bRefreshAll` a new stack restarts the life of every stack.
- **What it does by itself.** `PerStackModifiers` are attribute changes on the holder, applied as one set scaled by the stack count and rebuilt when the count changes: five stacks of -6 % More are -30 %, not 0.94 to the fifth. `BossScale` multiplies them on a Boss-tier holder. A meter with `bAbsorbsHits` takes incoming damage before Health does.
- **Events.** Gaining, reaching the cap, spending and running out are told to the web of whoever applied the state (`OnStateGained`, `OnStateAtCap`, `OnStateSpent`, `OnStateExpired`), with the amount and the state as the noun. A draining meter tells nobody of each drop; it expires when it is empty.

A state on another pawn takes the **applier's** resolved copy of its row (a node may change its cap, life or decay through a row override); a self state takes the holder's; a shared pile takes the copy of whoever adds to it.

## Ailments

An ailment is a row of `DT_KitAilments`. Six today:

| Ailment | Type | Duration | Tick | Front load | Cap per source |
|---|---|---|---|---|---|
| Poison | Poison | 4 s | 0.5 s | 0 | none |
| Bleed | Physical | 6 s | 0.5 s | 0.3 | 3, the weakest replaced |
| Burn | Fire | 4 s | 0.5 s | 0 | none |
| Blight | Poison | 8 s | 1 s | 0 | 3, the oldest replaced |
| Plague | Poison | 8 s | 1 s | 0 | none (run by its own mechanic) |
| Coil | Physical | 6 s | 1 s | 0 | 5, the oldest replaced |

- **A share of the hit.** Each application is one independent **instance** worth a share of the hit that applied it, read before the target's mitigation, raised by the applier's `AilmentDamageIncreased` line, and paid out over the row's duration (times the move's duration scale and the applier's `AilmentDuration` line). The front load is owed at once; the rest is spread evenly. A row or op may give a flat total instead when the hit itself deals none.
- **Summed ticks.** Every instance accrues on its own clock. When a tick is due, each source's instances of one ailment on one target pay as **one summed hit**, never one hit per instance. A tick is a Triggered secondary hit: only the target's mitigation changes it, with no hit roll and no crit, and it fires no `OnHit`, `OnCrit` or `OnHitTaken`.
- **The cap.** With `CapPerSource` above 0, a source at its cap on a target replaces its weakest instance (`bReplaceWeakest`) or its oldest. With replace-weakest, a new instance weaker than the weakest is dropped. The dropped instance is told as `OnAilmentDisplaced`.
- **Only a Direct hit plants** a move row's own ailment (the E-2.166 rule, as the legacy poison).

**The verbs** (ops; `Kits/KitOpsBuiltin.cpp`, `Kits/KitOpsExtra.cpp`, `Kits/Mechanics/KitAilmentVerbs.cpp`):

| Op | What it does |
|---|---|
| `ApplyAilment` | One instance on the target: `ShareOfHit`, or `Flat` from any trigger. |
| `ConsumeAilment` | Ends the user's instances on the target and deals what was left at once, as a final number; `Share`, a splash `Radius`, `HealShare`. |
| `ExtendAilment` | Live instances run longer at the rate they had, so they pay more in all. |
| `SpreadAilment` | Copies live instances to enemies near the target at `Share` of what is left, for the time left. A copy never copies. |
| `AccelerateAilment` | The same total, paid faster. |
| `TickAilment` | Deals so many seconds of ticks now, on top of what the instances still pay. |
| `ConvertAilment` | The instances end and one instance of another ailment takes their place, applied as a copy. |
| `MoveAilment` | Answers `OnAilmentDisplaced`: the dropped instance goes to the nearest other enemy with room, as a copy. |
| `TapAilment` | Answers `OnAilmentApplied`: the user heals a share of the instance over its life; half of the rest at once when it ends early. |

Events told to the applier's web: `OnAilmentApplied`, `OnAilmentConsumed`, `OnAilmentExpired`, `OnAilmentDisplaced`. The rule set `AilmentRules` lets a node raise new instances (`Increased`) or the time an extension adds (`ExtendedMore`).

## The status row

`UStatusRowWidget` reads three replicated lists on the wrapper ASC and nothing else: the plate states (states on a pawn that everyone sees), the self states (replicated for the holder's status row), and the ailment views (per ailment: the live instance count and when the last one ends). A row's `bShowOnHud` and `bShowOnPlate` decide where a state appears. The lists are rewritten when something comes or goes, not as every clock runs down; the widget rebuilds its entries at most five times a second and the paint only works the rings out from the clock.

Numbers in `Config/DefaultGame.ini` (`UHudLayoutSettings`): `StatusDiscSizePlate` 18, `StatusDiscSizeHud` 34, `StatusMeterWidthInDiscs` 3.0, `StatusRowMaxEntries` 10, `StatusRowRefreshSeconds` 0.2.

Entrance (an AI control, see [Minions and monsters](kit-minions.md)) is its own state on the field monster, not one of these lists, so no plate shows it yet.

## The caps page

Rows of `Data/DT_CombatRules.csv` that hold whatever a node says. The caps and late verbs do not act with `Slime.SoulWebs 0`. The quotes behind them are section 4 of `docs/plans/soul-redesign-2026-10-03.md`.

| Row | Value | What it does | Read by |
|---|---|---|---|
| `DodgeCap` | 0.75 | Ceiling of the `Dodge` attribute | `Abilities/MonsterAbilitySystemComponent.cpp` |
| `BlockChanceCap` | 0.75 | Ceiling of `BlockChance` | same |
| `BlockValueCap` | 0.6 | Most of a hit a block stops | same |
| `DamageTakenLessCap` | 0.9 | Ceiling of `DamageTakenLess` | same |
| `DefenseMitigationCap` | 0.8 | Defense never removes more than this share of a physical hit | same |
| `RootDuration` | 1.5 | A root's length when the row names none | same |
| `PullDistance` | 300 | A pull's distance when the row names none | same |
| `HealthPoolCap` | 0.5 | Ceiling of a `bAbsorbsHits` meter, of its holder's maximum health | `Abilities/MonsterAbilitySystemKit.cpp` |
| `AreaOverlapShare` | 0.3 | Of one owner's placed things of one row a pawn stands in together, the oldest lands in full and every other at this share | `Kits/KitOpsLate.cpp`, `Kits/Mechanics/FaultlineRouting.cpp` |
| `StatScaleCeiling` | 1 | Ceiling of a `ScaleByStat` with none of its own; a flat is also held to the hit's own flat | `Kits/SoulWebComponent.cpp`, `Kits/Mechanics/KitMech3.cpp` |
| `RedirectCap` | 0.4 | Most of a hit `RedirectDamage` rules may move to minions or a decoy | `Kits/KitOpsLate.cpp` |
| `HardControlWindow` | 8 | Seconds of the hard-control clock | same |
| `HardControlBudget` | 4 | Seconds of new stun, freeze and root, counted together, a monster is granted inside the window; a refresh counts what it adds | same |
| `EliteHardControlScale` | 0.5 | An Elite's hard control is scaled by this | same |
| `PlayerHardControlScale` | 0.5 | A player's hard control is scaled by this | same |
| `PlayerHardControlCap` | 1 | And held to this many seconds | same |
| `PlayerHardControlImmunity` | 3 | No hard control lands on a player for this long after one ends | same |
| `DisplacementImmunity` | 2 | A player knocked back or pulled is not displaced again for this long | same |
| `SoftControlMagnitudeCap` | 0.9 | Ceiling of a slow, weaken, shock or blind after dealt-magnitude lines | same |
| `MinionBudget` | 4 | Minions one owner holds, before the `MinionBudget` attribute | `Kits/KitMinions.cpp`, `Kits/KitRules.cpp`, `Kits/Mechanics/Bewitch.cpp` |
| `MinionAggroRange` | 900 | A minion takes an enemy within this of its owner | `Monsters/MonsterStateTree.cpp` |
| `MinionLeashRange` | 1500 | And keeps it while within this of its owner | same |
| `MinionTeleportRange` | 2500 | Past this it is placed beside its owner | same |
| `MinionFollowDistance` | 250 | How close it follows | same |
| `MinionSummonRadius` | 140 | Where summoned minions are placed round the spot | `Kits/KitMinions.cpp` |
| `MinionCommandSeconds` | 6 | How long an attack order holds | `Kits/KitMinions.cpp`, `Kits/Mechanics/Bewitch.cpp` |
| `MinionReturnSeconds` | 2 | How long a return order holds | `Kits/KitMinions.cpp` |
| `MinionLeapRadius` | 160 | Radius of a commanded leap's landing hit | `Kits/KitMinions.cpp`, `Kits/Mechanics/Bewitch.cpp` |
| `RemainsRadius` | 1500 | A death this near a Remains player leaves it a Remains | `Kits/Mechanics/Remains.cpp` |
| `PressBudget` | 12 | Different ops one press may run (one op is one effect, however many pawns or hits; any other event has as many of its own) | `Kits/KitRules.cpp` (`FKitBudget::Size`) |
| `StaticArcHitCap` | 10 | Most arc hits one release lands | `Kits/Mechanics/StaticArc.cpp` |
| `PlagueHostCap` | 24 | Most hosts one bite's lineage reaches | `Kits/Mechanics/Plague.cpp` |
| `PlagueGenerationCap` | 5 | Most generations of one lineage | same |
| `StokeHeatCap` | 1000 | Ceiling of the Stoke's heat meter | `Kits/Mechanics/Stoke.cpp` |
| `SwallowHoldCap` | 8 | Longest a swallowed monster is held, seconds | `Kits/Mechanics/Swallow.cpp` |

`ManaCostLess` is capped at 0.9 and `SelfDamageLess` at 0.9 in code, not by a row.

Two more rules of the same page are not numbers: a secondary effect is a share of its parent hit and takes no gem flat; consumed, converted, reflected and self-inflicted damage is final (nothing but a rule that names it changes it).

## For testers

Host only. `Slime.State.Add Sunder 3 FieldMonster_0` adds stacks from the local player; `Slime.State.Show [Pawn]` lists the piles and the replicated lists; `Slime.Ailment.Show [Pawn]` lists instances, time left and damage still to pay (the last on the authority only). `Slime.Web.Rows State Sunder` prints the row as your nodes resolve it beside the table's own. `Slime.Kit.Late [Pawn]` prints the hard-control clock and immunities.

Log lines: `State: <pawn> <state> 2 -> 3 of 5 from <applier>`, `State: <pawn> spent 3 of <state> (0 left)`, `State: <pawn>'s <state> absorbs 12.00 (38 left)`, `Ailment: <pawn> +1 <ailment> from <source> (24.00 over 4.0 s; 2 from that source, 2 live)`, `Ailment: <pawn>'s <ailment> from <source> consumed with 18.00 left`, `Kit: no <table> row named '<name>'; nothing applied (said once)`.

## What is not built

- Icons on the discs; the row's `Icon` column is not drawn.
- A plate for an entranced monster.
- The legacy poison stacks and the kit ailment Poison are two systems side by side; `Slime.Ailment.Show` prints both counts.
- Nothing here is saved, and nothing has been watched in a joiner's window by a person.

## For engineers

State and ailment storage is on the wrapper ASC, beside the counts, the poison stacks and the controls, so there is no fifth wrapper type (CLAUDE.md 6.8): `Source/SandboxARPG/Abilities/MonsterAbilitySystemKit.cpp` and the kit block of `MonsterAbilitySystemComponent.h` (`AddKitState`, `SpendKitState`, `GetKitState`, `ExtendKitState`, `AbsorbWithKitMeters`; `ApplyKitAilment`, `ConsumeKitAilment`, `ExtendKitAilment`, `SpreadKitAilment`, `AccelerateKitAilment`, `TickKitAilmentNow`, `ConvertKitAilment`; the pure `Evaluate*` rules the tests call). Two timers at four ticks a second while anything is live; no per-frame work. The three replicated lists are `KitPlateStates`, `KitSelfStates` and `KitAilmentViews` (push model), with `OnKitStatusChanged` on every machine. The widget is `UI/StatusRowWidget.*`, built in code; `UI/NamePlateWidget.cpp` and `UI/PlayerHud.cpp` host it. The caps as pure rules are `FKitCaps` in `Kits/KitOpsLate.h`, enforced through `FKitLate::FilterControl`, `RedirectIncoming` and `PlacedOverlapShare`. Tests: `SandboxARPG.Kits.States.CapDecayAbsorbMath`, `Kits.Ailments.ShareFrontLoadTickMath`, `Kits.Caps.*`, `UI.SoulWeb.StatusDiscsCountAndDrain`.
