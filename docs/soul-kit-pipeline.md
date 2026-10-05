# Soul kit pipeline

**Status:** in place, provisional (register row `soul webs`). How a soul goes from a design draft to tables and assets the game reads. The contract is `docs/design/soul-kit-schema-v2.md`; the short form of the commands is `Data/Kits/README.md`. What the tables do in the game is [Soul webs](soul-webs.md).

## The idea

Nothing about a node is code. A soul is written as prose once (the draft), then mapped once into the schema's vocabulary (the sidecar). A generator turns the two into table sources and one name-only class per ability. A lint says what is mapped and what is not. Two commandlets put the tables and assets into the project. A node the vocabulary cannot express is still a row: it can be taken and read, and it says on the screen that nothing acts.

## The sources (edit these)

All under `Data/Authoring/Souls/`.

| File | What it holds |
|---|---|
| `<soul>.json` | The **draft**, frozen design input: `fantasy`, `mechanics`, `builds`, `basicAttack`, `actives` (six, each with `id`, `name`, `base` text and `nodes`: `id`, `name`, `defining`, `requires`, `excludes`, `stages` one text per rank), `passive` (`centre`, 39 `slots` keyed `r1a0` to `r4a300` with `name`, `type`, `price`, `stages`; `excludes` pairs). The same content as `docs/design/souls/<soul>.md`. |
| `<soul>.map.json` | The **mapping sidecar**: `abilities` (per ability id: `class`, `kind`, `tags`, `clip`, `hitDelay`, `row` of move columns, `cues`, optional `ops`, `built`, `note`), `basic`, `soulOps`, `nodes` (keyed `<ability_id>/<node_id>` or `passive/<slot>`: `overrides`, `modifiers`, `ops`, `rowOverrides`, `opOverrides`, `built`, `note`), the soul's own `states`, `ailments` and `placed` rows, and `monster` (the move ids a field monster of this soul uses). |
| `shared.map.json` | Rows more than one soul uses: 3 ailments, 1 state (`Sunder`), 1 placed row (`Crater`) and the 24 cues. |
| `bodies.json` | Per soul: mesh, capsule, mesh offset and the clip table (`idle`, `walk`, `run`, `hit`, `death`, `attack_light`, `attack_heavy`, `attack_alt`, `cast`, `roar`, `dash`). A key a body lacks falls back to `attack_light`, then `idle`. `keepAsset` leaves the body of an existing asset as it is (Creepy, Corpse). |
| `vocabulary.json` | Written by the engine side: every trigger (70), condition (80), op (124), target (9), move column (87), attribute (62) and stamped event (10), each `built` true or false with a note. Without the file the lint reads the lists of the schema document. |
| `monsters.json` | The ten souls as field monsters: the picker's numbers per kit, which `DA_Monster_*` uses which kit, and which two souls each field takes. See [Minions and monsters](kit-minions.md). |
| `Art Assets/UI/Icons/Abilities/Kit/T_Icon_Kit_<Soul>_<Ability>.png` | Ability pictures; a missing one is skipped. |

Rules of a sidecar the lint holds you to: numbers come from the draft text rank by rank; a percent in the text is a fraction (12 % is `0.12`); a list shorter than the node's ranks repeats its last value; state, ailment and placed names are global (one name, one meaning; a name defined twice with different content is an error); `built` is `full` only when everything the text says happens at every rank, never for an approximation.

## The generator

`python scripts/gen_soul_webs.py` writes; `--check` exits 1 if a write would change anything. Any other argument, `--help` included, is ignored and the script writes. It reads the drafts, the sidecars, `shared.map.json` and `bodies.json`, parses `Source/SandboxARPG/Kits/KitTypes.h` so a new property there appears in the output at its default, and writes under `Data/Kits/`:

- `DT_KitNodes.json`, `DT_KitAbilities.json`, `DT_KitStates.json`, `DT_KitAilments.json`, `DT_KitPlaced.json`, `DT_KitCues.json`: one object per row.
- `DT_MoveStats_Kit.csv` (one row per v2 ability) and `DT_MoveStats_Merged.csv` (the legacy rows of `Data/DT_MoveStats.csv` followed by the kit rows, every row every column; this is what fills `DT_MoveStats`).
- `SoulAssets.json`: what goes on each `DA_Soul_<Soul>` (body, clips, moveset).
- Under the `// GENERATED BELOW` marker of `Source/SandboxARPG/Kits/KitAbilities.h` and `.cpp`: one name-only `UKitAbility` subclass per ability, `Kit<Soul><AbilityPascal>` (the basic attack is `Kit<Soul>Basic`). The class name is also the `DT_MoveStats` row name. The hand-written top of those files is never touched.

What it works out for you: the six Unlock rows (they carry the ability's own `ops`) and the centre row `<Soul>.Passive.centre` (the soul's `soulOps`, then the basic attack's ops); the passive graph's neighbours and positions from the slot names; an active-web op's `TriggerAbility` (its own ability unless `"triggerAbility"` says `Any` or another class); any-of groups flattened into `FKitCondition::Group`; enum values written by name turned into their index; an `opOverrides` entry resolved to the Unlock or centre row and the op's index. `HitRange`, `HitArc`, `HitDelay` and `ProjectileRange` written in a sidecar `row` go to the soul asset's move, not to `DT_MoveStats`; otherwise `HitRange` and `HitArc` come from `ShapeLength` and `ShapeAngle`.

Today it writes 10 souls, 968 node rows and 70 ability rows.

## The lint

`python scripts/check_soul_webs.py` (add `-v` for every warning) reads what the generator reads, writes nothing, and exits 1 on any error.

- **Errors:** a column, trigger, condition, op, target or attribute outside the vocabulary; an undefined state, ailment, placed row or cue; a values list longer than the node's ranks; a `full` node with nothing on it; an `opOverrides` index that names no base op.
- **Warnings:** a node or ability with no sidecar entry; a param the schema does not list for its op; an op with no target; a trigger that takes a noun and has neither `triggerName` nor a matching `name` (it fires for any); a summon name that is no row of `Data/DT_MonsterStats.csv`.

Then the coverage table, one line per soul:

| Column | Meaning |
|---|---|
| `nodes` | Draft nodes (the six Unlock rows and the centre are not counted). |
| `full`, `partial`, `none` | How many nodes claim each. |
| `abil`, `partial`, `none` | Abilities full, then partial and none (seven per soul with the basic attack). |
| `v2.1 n/a` | Node rows and abilities that use a word or structure only schema section 7 has. |
| `engine-unbuilt` | Full or partial nodes that use a word whose `vocabulary.json` entry is `built: false`. |

The current numbers are on the [Soul webs](soul-webs.md) page: 898 nodes, 370 full, 383 partial, 145 none; 0 errors, 45 warnings.

`python scripts/check_souls.py` is the older check of the drafts' own shape (slot names, the passive graph) and runs first.

## The commandlets

Both run with the editor closed, from PowerShell (Git Bash rewrites `/Game` paths), against the editor DLL of the checkout they run in, so that checkout is built first.

```
python scripts/check_souls.py
python scripts/gen_soul_webs.py
python scripts/check_soul_webs.py
# build SandboxARPGEditor of this checkout, editor closed
# lock: Content/Data/DT_MoveStats.uasset, Content/Souls/DA_Soul_Creepy.uasset, Content/Souls/DA_Soul_Corpse.uasset
$env:SOUL_WEBS_DRY = "1"
& "D:\UE_5.8\Engine\Binaries\Win64\UnrealEditor-Cmd.exe" "<checkout>\SandboxARPG.uproject" -run=pythonscript -script="<checkout>\scripts\editor\soul_webs.py" -unattended -abslog="<log>"
Remove-Item Env:SOUL_WEBS_DRY      # then the same command for the real run
python scripts/gen_soul_webs.py --check
python scripts/check_asset_names.py
```

**`scripts/editor/soul_webs.py`** reads only `Data/Kits/` and `Art Assets/`. It creates the six `/Game/Data/Kits/DT_Kit*` tables when missing and fills them, refills `/Game/Data/DT_MoveStats` from the merged CSV, imports the ability pictures into `/Game/Art/UI/Icons/Abilities/Kit/`, and writes `/Game/Souls/DA_Soul_<Soul>` for the ten souls (body, clips, growth, orb icon, the moveset: the basic attack, then the six abilities). On the Creepy and the Corpse the legacy moves are kept after the v2 ones and the legacy `Tree` is neither set nor cleared, because the legacy field monsters and `Slime.SoulWebs 0` still use them.

| Switch | Effect |
|---|---|
| `SOUL_WEBS_DRY=1` | Validates inputs, classes and assets; writes nothing. |
| `SOUL_WEBS_DROP_LEGACY_MOVES=1` | Drops the legacy moves from the Creepy's and Corpse's moveset. |
| `SOUL_WEBS_REIMPORT_ICONS` (set) | Re-imports pictures already imported. |

Read the log for `SOUL_WEBS:` lines: one `VERIFY <table>: N rows in the asset, N in the source, N missing, N extra, order ..., N mismatch(es) over ..., saved ...` per table, `VERIFY icons: ...`, one `VERIFY DA_Soul_<Soul>: ...` per soul (moves, how many v2 and legacy, clips found, capsule), and `done: N error(s)` at the end. Locks wanted (CLAUDE.md 5.3): `DT_MoveStats`, `DA_Soul_Creepy`, `DA_Soul_Corpse`; everything else it writes is a new file.

**`scripts/editor/soul_monsters.py`** runs after it (the ten soul assets and the `UKit*` classes must exist). It refills `/Game/Data/DT_MonsterStats`, `DT_SpawnTable` and `DT_CombatRules` from `Data/*.csv`; writes `/Game/Monsters/DA_Monster_<name>` for every entry of `monsters.json` (a new one is a copy of `DA_Monster_Corpse`, then its soul, attack move, move list, display name and body scale are set); and re-points the grey-box spawners of Zones 1 to 4 on `/Game/Levels/Lvl_Slice_WP` to their field's souls.

| Switch | Effect |
|---|---|
| `SOUL_MONSTERS_DRY=1` | Changes nothing; lists what would change and writes the spawners' external actor files to `Saved/SoulMonsters/spawner-files.txt` (the files to `git lfs lock`). |
| `SOUL_MONSTERS_SKIP_MAP=1` | Tables and definitions only. |

Its lines start `SOUL_MONSTERS:`, with one `VERIFY` per table, per definition and for the map. Locks wanted: the three tables, every existing `DA_Monster_*` it edits, and the spawner files of the dry run.

Both scripts are run by a person or on a person's word, under lock (CLAUDE.md 5.3, 7.8).

## How to add things

- **A node.** Add it to the draft and give it a sidecar entry under `nodes`. Generate, lint, run `soul_webs.py`. No build is needed when only table rows changed; `Slime.Web.Reload` reads the tables again in a running game.
- **An ability.** Add it to the draft's `actives` and to the sidecar's `abilities` with its `class` and `row`. The generator writes a new class under the marker, so build before the commandlet. Add its id to `monster.moves` and to `monsters.json` if field monsters should use it.
- **A state, ailment or placed row.** Add it to the sidecar's `states`, `ailments` or `placed`, or to `shared.map.json` when more than one soul uses it.
- **An op, condition or trigger.** One registrar object in a `.cpp` of `Source/SandboxARPG/Kits` (`FKitOpRegistrar`, `FKitStandingOpRegistrar`, `FKitConditionRegistrar`, `FKitTriggerRegistrar`, `FKitTargetRegistrar`; a trigger also needs its `Dispatch` call where the event happens). Add the word to `vocabulary.json` with `built: true` and to the schema document's section 7.8. `SandboxARPG.Kits.Vocabulary.BuiltNamesHaveHandlers` fails when a built name has no handler.
- **A move column.** A property on `FMoveStatRow` (`Abilities/MoveStats.h`), the code that reads it, the generator's column list, and `vocabulary.json`.
- **A soul.** A draft, a sidecar and a `bodies.json` block, plus its key in the `SOULS` list of both `scripts/gen_soul_webs.py` and `scripts/check_souls.py` (the list is typed in each). For field monsters: a `monsters.json` kit and definitions, and rows in `Data/DT_MonsterStats.csv` and `Data/DT_SpawnTable.csv`.

## Traps the scripts record

- `DT_KitNodes.json` is keyed `RowName`, not `Name`: `FKitNodeRow` has a display-name property and a table's JSON key field is `Name` by default. The commandlet sets the table's import key field before the fill and puts the default back on the others.
- The commandlet runs the checkout's own DLL. A v2 column `FMoveStatRow` does not have in the build is left out of the fill and reported as an error; the import does not fail.
- A struct property cannot be set on an instance from editor Python; the script builds a fresh struct with constructor arguments. `unreal.Rotator` is (roll, pitch, yaw).
- A read-only file means its LFS lock is not held. `soul_webs.py` makes it writable and names it in the log; `soul_monsters.py` names it and does not write it.
- Every picture re-import rewrites the texture's bytes, which would churn sixty LFS files per run; that is why imported pictures are left alone by default.
- `soul_webs.py` loads no map. `soul_monsters.py` does, and leaves a `RecastNavMesh` external actor package behind: delete it before a commit.
- Whether an op of an ability web hears an event is decided by the dispatcher (`FKitRules::HearsEvent`; schema 4.3 has the three lists). An event that always belongs to one ability of the user (`OnUse`, `OnHit`, `OnCrit`, `OnDashEnd`, `OnSwingEnd`, the phase events, `OnBuffEnd`, `OnHitOwnPlaced`, and the rule sets `ArcRules`, `StokeRules`, `PlagueRules`, `SwallowRules`, `CurseRules`, `OverrideRules`) is heard only by that ability's web. An event that never carries one (`OnInterval`, `OnStill`, `OnHitTaken`, `OnBlock`, `OnHealthBelow`, the state, ailment and control events, `LeechRules`, `ThresholdRules`, `AilmentRules`, `RemainsRules`) is heard by every op with no `triggerAbility` at all. An event that sometimes carries one (`OnKill`, the placed and minion events, the mechanics' own) is heard when it carries the op's own ability or none. So `"triggerAbility": "Any"` is needed only to hear **another** ability's events of the first and third kind (an `OnCrit` op of the Drum's web that counts every ability's crits); on an op of the second kind it is harmless. Standing rules are not events: a `BonusDamage` or `ScaleByStat` of an ability web applies to that ability's hits unless it says `Any` or names another ability.
- `EnemiesNear`, `EnemiesNearWithState`, `EnemiesNearWithAilment` and `OwnPlacedNear` read `values` as `[count, radius]`, not per rank.
- The lint warns that summons `Bonewalker` and `Sapling` are no rows of `Data/DT_MonsterStats.csv`. The rows are there as `DA_Monster_Bonewalker` and `DA_Monster_Sapling`, and the summon code takes either spelling.

## For engineers

`scripts/gen_soul_webs.py`, `scripts/check_soul_webs.py` (imports the generator as a module), `scripts/check_souls.py`, `scripts/editor/soul_webs.py`, `scripts/editor/soul_monsters.py`. Row structs: `Source/SandboxARPG/Kits/KitTypes.h`. Table paths: `[/Script/SandboxARPG.KitTables]` in `Config/DefaultGame.ini`. `python scripts/gen_soul_webs.py --check` is the test that the generated files match their sources. Provisional markers: register row `soul webs`.

## Feel defaults: play rate and commit window

The bodies' clips are two to five seconds long, and a kit move with no `PlayRate` and no `CommitSeconds` held the pawn for the whole clip. The generator now fills both for every row whose sidecar leaves them out:

- `Data/Authoring/Souls/clips.json` holds each clip's length. `scripts/editor/dump_clip_lengths.py` writes it by commandlet from `bodies.json`; run it again when a body's clip table changes.
- `Data/Authoring/Souls/feel.json` holds the swing time per kind of move (basic 0.8 s, attack and spell 1.0 s, stance and command 0.8 s), the recovery after the last hit (0.25 s) and the play rate's bounds. A draft text that states its own time ("0.8 s swing", "0.6 s cast") wins over the kind's number, and a sidecar ability's own `"swing"` wins over both.
- `PlayRate` becomes clip length over swing time. `HitDelayScale` is written at the same rate, so the sidecar's `hitDelay` and `ExtraHitDelays` stay the seconds they were written as; attack speed, a slow or a node's `PlayRate` override still moves them.
- `CommitSeconds` becomes the last hit's time plus the recovery (a dash: its `DashSeconds` plus the recovery; a move with no hit: 0.3 s), never longer than the swing.
- `CommitRate` is written with a derived `CommitSeconds` (not a dash's, whose window is its `DashSeconds`): the rate the window was derived at, so it shrinks as attack speed raises the move's rate. A window a sidecar wrote has none and stays real seconds.
- A `PlayRate` or `CommitSeconds` written on a sidecar row is kept. Channel rows are left alone.
- The bounds in `feel.json` are the generator's (the rate it may derive). At runtime the rate has no ceiling; the clip on the mesh is held between the `MoveClipRateMin` and `MoveClipRateMax` rows of `Data/DT_CombatRules.csv`, which `scripts/editor/soul_monsters.py` refills.

Every number is a placeholder for Jon's feel pass. Field monsters wear the same rows, so their swings are faster too.

## Clip contact times (2026-10-05)

A move's hit lands by timer; its clip is a picture of it. So that the picture shows the hit when the hit lands, each clip a
kit move plays has a measured contact time, and the clip is started at an offset on the mesh. Nothing mechanical reads it:
the hit time, the commit window and the re-fire time are what they were.

- **Measure**: `scripts/editor/dump_clip_contacts.py` (a pythonscript commandlet, run like `dump_clip_lengths.py`; no map,
  read-only) samples every clip named in `Data/Kits/SoulAssets.json` 60 times a second in component space
  (`unreal.AnimPoseExtensions`) and takes, of the hand, weapon, head, foot and tail bones, the one that reaches the highest
  speed; the contact is the time of that peak speed. A clip whose fastest bone never passes 150 units a second or twice its
  own mean speed has no clear strike and gets no entry. The result is `Data/Authoring/Souls/contacts.json` (contact, the
  bone, its speed, and the time of its furthest reach for comparison). Run it again when a body's clip table changes.
- **Data**: the generator writes `clipContact` on each move of `SoulAssets.json` (0 for a clip with no entry and for the
  idle, walk and run clips standing in for a move's), and `scripts/editor/soul_webs.py` sets it as `FSoulMove::ClipContact`
  on the soul asset, beside the clip it belongs to. 0 is the old behaviour, so the legacy souls' own entries are untouched.
- **A plain swing** (`USoulComponent::PlayMove`): the clip starts at the contact less the hit delay times the play rate
  (`FKitPhaseRules::ClipStart`). When the contact comes sooner than that, the clip starts at its first frame and the mesh
  plays it slower (contact over hit delay), while the windows keep the move's rate. A dash is left alone.
- **A wind-up** (the `WindUp` phase in `Kits/KitPhases.cpp`, on every machine by the replicated phase): the clip starts at
  the press. When the swing would start part way into the clip, the wait plays the clip up to that frame and the clip
  multicast goes on from it at the move's rate; otherwise wait and swing are one slow play from the first frame, which the
  multicast lets run on, the contact landing at the wind-up's end. A wind-up a held key stretches (`MaxHold`) keeps the
  held first frame, since its end is not known at the press.
- Known: a slowed clip is cut by locomotion when the swing's window ends, part way through its recovery. The log line
  `move clip ... shown from X s at xR` and `KitPhase: ... animates the wind-up of ...` say what the mesh was given.
