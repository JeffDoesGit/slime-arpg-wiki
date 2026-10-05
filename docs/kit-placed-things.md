# Kit placed things

**Status:** first pass, provisional (register row `soul webs`). Pools, lines, bells, seeds, rubble, decoys: anything a move of a v2 soul leaves on the ground. Every number is a column of the thing's row in `DT_KitPlaced` and a placeholder. Part of [Soul webs](soul-webs.md).

## What you see

A flat disc or strip on the ground in the row's colour, brighter once it is ripe, with the row's loop effect when it names one. Enemies that walk in are hurt, slowed or marked; some things pulse, some burst when they end or when a move sets them off, some fly to where you aim, some follow you. The flat shapes are placeholders for the effects the VFX review lists.

## The store

There is **never an actor per thing and never a timer per thing**. One manager per world (`UKitPlacedSubsystem`) holds every placed thing as a plain struct and steps all of them on one timer, four times a second. Authority only. What clients see is one replicated list per owner (below).

A thing is placed by a move whose row names `PlacedName` (with `PlacedCount` and `bPlaceAtSelf`), by the `Place` op, or by a mechanic. Several placed together go on the spot first, the rest on a ring twice the row's radius out. Placing does nothing for a dead owner, for a row that does not exist (warned once), or in a safe zone. A dead or gone owner's things go, with no burst and no event.

## Rows

A row of `DT_KitPlaced` (`FKitPlacedRow`; 28 today: Crater, BlightBloom, Rubble, Faultline, Bell, Seed, Remains and the rest):

| Columns | Meaning |
|---|---|
| `Shape`, `Radius`, `Length` | A circle of `Radius`, or a line: a segment from where it was placed toward the aim, `Length` long and `Radius` wide each side, with round ends. All flat; height is ignored. |
| `Life` | Seconds it lasts; 0 never expires (still bound by the caps). |
| `CapPerOwner` | How many one owner may hold; placing one more removes the oldest of that row. |
| `PulsePeriod`, `PulseDamage`, `DamageType` | A pulse on everything inside every period; 0 never pulses. |
| `EnterShare` | An enemy that walks in is hit once for `PulseDamage` times this. |
| `BurstShare`, `bBurstOnExpire` | A burst hits everything inside for `PulseDamage` times this; with the flag, the end of its life is a burst. |
| `Ailment`, `AilmentShare`, `AilmentFlat` | The ailment a pulse, enter hit or burst applies, as a share of its damage or a flat total when the pulse deals none. |
| `ApplyState`, `StateStacks` | The state a pulse or enter adds to each enemy inside. |
| `ControlKind`, `ControlMagnitude`, `ControlDuration` | The control each pulse or enter applies inside. |
| `OwnerInsideModifiers` | Attribute changes on the owner while it stands inside one of its own. |
| `bFollowOwner` | It rides on its owner. |
| `Health`, `TauntRadius` | Above 0 the thing can be hit and dies at 0 (a decoy); monsters within the radius go for it. |
| `RipeSeconds` | Seconds until it counts as ripe; 0 is ripe at once. |
| `PulseStepIncreased`, `PulseStepCap` | Each pulse lands that much increased over the one before, for so many pulses. |
| `bOffBudget` | Outside the per-player budget (a Remains); its own cap still holds. |
| `Color`, `Letter`, `CueLoop`, `CuePulse`, `CueBurst` | How it is drawn and what it plays. |

The row a thing uses is its **owner's resolved copy**: a held node may change a field through a row override. The owner's `PlacedSize` and `PlacedDuration` attribute lines are read once, when the thing is placed: the radius and the life are multiplied by 1 plus the line.

## What happens each step

- **Entering.** An enemy standing inside when a thing is placed has entered it, at the first step. An entering enemy takes the enter hit and the row's state and control; `OnEnemyEntersPlaced` is told.
- **Pulses.** When a pulse is due, everything inside takes the row's whole pulse. The next pulse is counted from the one that was due, so the period does not drift. `OnPlacedPulse` is told with the pulse's number.
- **Ripeness.** At its ripe time the thing is told as `OnPlacedRipe`, once.
- **Dying inside.** A pawn that dies in a thing tells its owner `OnEnemyDiesInPlaced`, for a kill by anyone, once per death per owner and row however many of that owner's things of the row it stood in.
- **Decoys.** A thing with `TauntRadius` asks the taunt seam each step, so the lure holds while it stands. `DamagePlaced` hurts it; at 0 it ends and `OnPlacedExpired` is told.
- **Followers and flights.** A follower moves with its owner. A thing in flight has nothing inside; where it lands it bursts at its flight's share.
- **Expiry.** At the end of its life it runs out (`OnPlacedExpired`), or bursts when the row says so.
- **Owner inside.** The owner's inside modifiers are applied while it stands in one of its own things of that row.

**The hits.** A pulse, an enter hit and a burst are area hits of the owner, Triggered, one link past what placed the thing. They take the owner's increased and more lines and no gem flat, and they carry the ability that placed the thing, so the web's ability-limited rules answer them.

**Overlap.** Of one owner's things of one row a pawn stands in together, the oldest lands in full and every other pulse, enter hit and burst at the `AreaOverlapShare` rule row (0.3).

## Bursts

**A burst ends the thing.** `Burst` hits everything inside for `PulseDamage` times the row's `BurstShare` plus the caller's extra share, tells `OnPlacedBurst`, and removes it. The `BurstPlaced` op's `Share` is added to the row's; `PulsePlaced`'s and `LaunchPlaced`'s multiply. `OnPlacedBurst` is told once for the burst itself (ops aimed at `Self`, `Aim`, `OwnPlaced` or nothing run there) and once for each pawn it touched (ops aimed at `HitTarget` or `EnemiesAroundTarget` run there).

The placed ops: `Place`, `BurstPlaced`, `PulsePlaced` (one pulse now; the thing's own clock is left alone), `LaunchPlaced` (the nearest thing flies to the aim, at param `Speed` or 1800 uu a second, and bursts there), `MovePlaced`, `ExtendPlaced` (what is left grows by seconds and a fraction, under a maximum; a thing that never ends is left alone), `HitAlongPlaced` (a hit along each placed line). `BurstPlaced`, `PulsePlaced`, `ExtendPlaced` and `HitAlongPlaced` act on the event's own thing when the event is about a thing of the row they name, else on every one the user holds.

**`OnHitOwnPlaced`** is told when a Direct hit of the user lands on a pawn standing in one of its things (the event carries the hit), when a dash ends in one, or when a use that makes no hit, projectile or dash of its own is aimed at one; once per use per thing, and never for the thing that same use placed. A hit or projectile that crosses a thing and lands on nobody is not seen.

**The `OwnPlaced` target.** In a placed event the aim is where the thing stands. A `Hit`, `Place`, `Summon`, `Pull`, `Cue` or placed op aimed at `OwnPlaced` runs once at the place (a cone or capsule `Hit` runs along a placed line); any other op acts on the enemies within its `Radius` param, else the thing's own radius.

## What the fix pass of 2026-10-04 changed

- A row's `PlacedCount` of 0 places none.
- A placed thing's pulse, enter hit and burst are the thing's own hits: the placing ability's row riders do not ride them, and an op of that ability's web answers them only when it carries `HitIsPlaced`. Passive ops with no ability hear them like any hit.
- A placed row with `bNoEnterHitAtPlace` gives no enter hit to an enemy that stood in it when it was laid (the cast's own hit is theirs); one that walks in later is hit.
- A lobbed projectile's row lays its thing where the lob lands. A use that lays a thing at the aim is held to the move's range.
- A swing whose footprint covers a thing of the user's own touches it (`OnHitOwnPlaced`) with or without a pawn in it. `LaunchPlaced` with `Distance` launches the struck thing along the user's facing.
- `OnPlacedPulse` carries how many enemies the pulse reached (`TargetsHitAtLeast`).
- A Seed asked to sprout in a safe zone is kept (not shown in a run).

## Budgets

Two limits, both ending the oldest with no event:

- The row's `CapPerOwner`, per owner and row.
- A per-pawn budget over every row: `PlacedBudgetPerPlayer` in `Config/DefaultGame.ini`, 16. No design source names the number. A row with `bOffBudget` neither counts toward it nor is ended by it.

## The replicated list and ground marks

Each owner's web component replicates one list, `PlacedList` (push model), written by the manager when the owner's things change: per thing its id, row, location, a line's yaw, the radius and length it was given when placed, the server time it ends and the server time it is ripe, whether it follows its owner, and the arrival time of one in flight. A client tells a moved thing from a new one by the id.

`AKitGroundMarks` is one transient actor per local player, spawned by the HUD, never replicated. On a timer (`GroundMarkRefreshSeconds`, 0.1) it draws every placed thing as flat pieces from a pool of engine basic shapes: a circle is one disc, a line one strip. It also draws the **telegraph** of a move in its wind-up, in the footprint of the move's own hit shape, so what is drawn is what will land: a circle one disc, a capsule a strip and a disc at its far end, a cone its two edges and `GroundMarkConeArcPieces` (6) strips closing the arc. It reads replicated state only, through `KitSeams::Placed()` and `KitSeams::Phases()`, and starts and stops each thing's loop cue. Numbers in `UHudLayoutSettings`: `bGroundMarks` on, opacity 0.3, ripe 0.6, telegraph 0.45, pool size 96, the material `M_Greybox`. A wind-up's telegraph resolves its move's row once and keeps it for the wind-up; it reads it again for a new wind-up, for new kit tables, or after `GroundMarkResolveSeconds` (0.5), so the timer reads no rule and copies no row at each refresh.

## For testers

`Slime.Placed.Show` lists every placed thing of the world (owner, row, place, size, time left, ripeness, enemies inside) and the budget. `Slime.Placed.Place <Row> [Distance] [Pawn]` places one that far ahead (default 300); `Slime.Placed.Burst [Row] [Pawn]` bursts what the pawn holds. `Slime.Web.Rows Placed <Row>` shows the row as your nodes resolve it. `Slime.Hud.Kit` logs the ground marks this machine draws. Authority only for the two that change something.

Log lines: `Placed: <pawn> places <row> #7 at (x, y) radius 250, life 6.0 s (1 of 3 of that row, 1 of 16 in all)`, `Placed: <pawn>'s <row> #7 bursts at (x, y) on 3 enemies at share 1.00 (...)`, `... #7 is ripe`, `... #7 runs out` (`and bursts`), `... #7 flies 600 uu to (x, y), lands in 0.33 s`, `... #7 takes 12.0 (38.0 left)`, `Placed: <pawn> holds its budget of 16 placed things; its oldest (<row>) goes`, `Placed: <pawn> is in a safe zone; <row> not placed`, `Placed: <pawn>'s <ability> touches its <row> #7`, `Placed: <victim> died in <owner>'s <row> #7`. Each pulse is logged at Verbose only.

## What is not built

- Effects: the discs and strips are placeholders, and a row's cues name systems the project already holds.
- A hit or projectile that crosses a thing without landing on a pawn tells nothing.
- Nothing placed is saved, and nothing here has been watched in a joiner's window by a person.

## For engineers

`Source/SandboxARPG/Kits/KitPlaced.*`: `UKitPlacedSubsystem` (a world subsystem: `Place`, `Burst`, `Pulse`, `Launch`, `Move`, `Extend`, `Remove`, `DamagePlaced`, the reads `GetOwned`, `IsInOwned`, `GatherInside`, and `Edit` and `TellFor` for the mechanic classes), `FKitPlacedThing` (the server struct; `Tag`, `Counter`, `Stamp` and `PulseCount` are what a mechanic keeps on a thing), `FKitPlacedRep` (the replicated entry), `FKitPlacedGeometry` (pure footprint tests, `Scatter`, `OverCap`, `BurstShare`). The ops are in `Kits/KitOpsExtra.cpp`; the overlap rule is `FKitLate::PlacedOverlapShare`. The list lives on `USoulWebComponent` (`PlacedList`, `SetPlacedList`, `OnRep_PlacedList`). `UI/KitGroundMarks.*` draws; `KitSeams.h` carries the read seam and the taunt seam. No tick, and nothing allocated per step once warm (CLAUDE.md 6.6). Tests: `SandboxARPG.Kits.Placed.*` (four), `Kits.OpsExtra.ThePlacedOpsRun`, `UI.SoulWeb.GroundMarkPiecesFollowTheShape`. Provisional markers: register row `soul webs`.

## The ops pass of 2026-10-04

- **`OnPlacedEvicted`.** A thing the row's `CapPerOwner` or the per-pawn budget pushes out when one more is placed tells its owner's web `OnPlacedEvicted`, once the new thing is in the store. `OnPlacedExpired` is still only a life that ran out (or a decoy's health); a node that means both lists both. Log: `Placed: <pawn>'s <row> #<id> gives way to #<id> (cap or budget); OnPlacedEvicted is told`.
- **`OnHitOwnPlaced` and `EventHasHit`.** The event is told for a landed hit on a pawn in the thing, for a swing's footprint over it, for a dash that ends in it and for a use with no hit of its own aimed at it. `EventHasHit` holds for the first two only.
- **`OnPlacedBurst`** is told once for the burst and once for each pawn it touched; an op whose target is the event's pawn runs in the per-pawn telling, every other op once.
- **A Seed's pop** is a placed thing's hit (`HitIsPlaced`), like a pool's pulse: the popping ability's `OnHit` ops answer it only by that condition.
- **An ailment a placed thing applies** tells `OnAilmentApplied` with the ability that placed the thing as its cause, so `HitAbilityIs` holds there.
