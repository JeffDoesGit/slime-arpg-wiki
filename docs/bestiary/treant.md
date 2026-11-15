# Treant

[Bestiary](README.md)

**Flavour.** You are a walking grove: slow, enormous and never in a hurry, because the ground under the fight belongs to you, and everything that grows there answers when you call.

![Treant, front](img/treant-front.png) ![Treant, side](img/treant-side.png)

## Clips its body plays

| Use | Clip | Seconds |
|---|---|---|
| idle | `AS_Treant2_Idle` | 1.67 |
| walk | `AS_Treant2_Walk_Aligned` | 2.80 |
| run | `AS_Treant2_Run` | 1.67 |
| hit | `AS_Treant2_HitReact` | 2.00 |
| death | `AS_Treant2_Death1` | 1.25 |
| attack light | `AS_Treant2_Attack1` | 1.67 |
| attack heavy | `AS_Treant2_Attack2` | 1.04 |
| attack alt | `AS_Treant2_Attack3` | 1.25 |
| cast | `AS_Treant2_Taunt` | 2.08 |
| roar | `AS_Treant2_Taunt` | 2.08 |
| dash | `AS_Treant2_Run` | 1.67 |

Note on the body: SK_Treant_2 is 478 uu tall (CT-2.47); at scale 0.6 that is 287, so half-height 143. Yaw -90 is the class default, unseen.

Its skeleton carries 12 clips in all (`Data/Authoring/Souls/body-clips.json`); the table lists the ones the soul names.

<!-- drops: written by scripts/gen_monster_catalogue.py from the data; edit the data, not this block -->
## What it drops

A kill at a time, by the zone it stands in; how to read it, and the odds zone by zone, are on the [bestiary index](README.md#what-drops). Every number is a placeholder, worked out from the tables the game reads.

| Monster | Zone | Class | Gems | Gem tier | Rarity | Colour | Bench items | Soul |
|---|---|---|---|---|---|---|---|---|
| Treant (`DA_Monster_Treant`) | Fields | Normal | 40 %, 0.40 a kill | II 80 %, I 20 % | Normal 60 %, Magic 30 %, Rare 10 % | Red 31.7 %, Blue 31.7 %, Green 31.7 %, Iridescent 5 % | Etching Needle 2 %<br>Expanding Lattice 3 %<br>Polishing Wheel 1.5 % | 20 % a Treant fragment; 3 make a soul (about 15 kills) |
| Elder Treant (`DA_Monster_Treant_Elite`) | Fields | Elite | certain, 1.40 a kill | III 80 %, II 20 % | Normal 25 %, Magic 50 %, Rare 25 % | Red 31.7 %, Blue 31.7 %, Green 31.7 %, Iridescent 5 % | Etching Needle 15 %<br>Expanding Lattice 20 %<br>Polishing Wheel 10 % | 100 % a Treant fragment; 3 make a soul (about 3 kills) |
| Treant (`DA_Monster_Treant`) | Jungle | Normal | 40 %, 0.40 a kill | III 80 %, II 20 % | Normal 60 %, Magic 30 %, Rare 10 % | Red 31.7 %, Blue 31.7 %, Green 31.7 %, Iridescent 5 % | Etching Needle 2 %<br>Expanding Lattice 3 %<br>Polishing Wheel 1.5 % | 20 % a Treant fragment; 3 make a soul (about 15 kills) |
| Elder Treant (`DA_Monster_Treant_Elite`) | Jungle | Elite | certain, 1.40 a kill | IV 80 %, III 20 % | Normal 25 %, Magic 50 %, Rare 25 % | Red 31.7 %, Blue 31.7 %, Green 31.7 %, Iridescent 5 % | Etching Needle 15 %<br>Expanding Lattice 20 %<br>Polishing Wheel 10 % | 100 % a Treant fragment; 3 make a soul (about 3 kills) |
<!-- drops end -->

Design sheet: `docs/design/souls/treant.md` in the repository. Mesh `SK_Treant_2`.

Pictures are frames from a game rendered off screen on 2026-10-28, the body standing idle in the town (grey-box map), HUD cropped away. Clip lengths are seconds at play rate 1, read from the clip assets (`Data/Authoring/Souls/clips.json` and `body-clips.json` for the souls).
