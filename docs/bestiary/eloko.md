# Eloko

[Bestiary](README.md)

**Flavour.** You are a knee-high thing in a coat of grass with too many teeth and one small bell: you never chase, you ring, and whatever hears it walks over to work for you or to be eaten.

![Eloko, front](img/eloko-front.png) ![Eloko, side](img/eloko-side.png)

## Clips its body plays

| Use | Clip | Seconds |
|---|---|---|
| idle | `ExoGame_MasterGrunt_Idle` | 4.10 |
| walk | `ExoGame_MasterGrunt_Walk_Fwd` | 1.40 |
| run | `ExoGame_MasterGrunt_Walk_Fwd` | 1.40 |
| hit | `AS_Eloko_Hit` | 0.50 |
| death | `ExoGame_MasterGrunt_Death` | 3.23 |
| attack light | `AS_Eloko_AttackLight` | 2.20 |
| attack heavy | `ExoGame_MasterGrunt_Attack_Melee_B` | 1.73 |
| attack alt | `ExoGame_MasterGrunt_Attack_Melee_C` | 2.10 |
| cast | `ExoGame_MasterGrunt_Attack_Ranged` | 2.83 |
| roar | `ExoGame_MasterGrunt_Cast_Summon` | 3.33 |
| dash | `ExoGame_MasterGrunt_Walk_Fwd` | 1.40 |

Its skeleton carries 23 clips in all (`Data/Authoring/Souls/body-clips.json`); the table lists the ones the soul names.

Design sheet: `docs/design/souls/eloko.md` in the repository. Mesh `SK_Master_Grunt`.

Pictures are frames from a game rendered off screen on 2026-10-28, the body standing idle in the town (grey-box map), HUD cropped away. Clip lengths are seconds at play rate 1, read from the clip assets (`Data/Authoring/Souls/clips.json` and `body-clips.json` for the souls).
