# Ogre

[Bestiary](README.md)

**Flavour.** You are the biggest thing in the room: every swing takes a second to arrive, nothing that hits you during it matters, and anything small enough goes in your mouth.

![Ogre, front](img/ogre-front.png) ![Ogre, side](img/ogre-side.png)

## Clips its body plays

| Use | Clip | Seconds |
|---|---|---|
| idle | `ExoGame_Troll_Idle_New` | 2.20 |
| walk | `ExoGame_Troll_Walk_Fwd` | 1.60 |
| run | `ExoGame_Troll_Walk_Fwd` | 1.60 |
| hit | `ExoGame_Troll_Hit_React` | 1.23 |
| death | `ExoGame_Troll_Death_` | 3.07 |
| attack light | `AS_Ogre_AttackLight` | 2.47 |
| attack heavy | `AS_Ogre_AttackHeavy` | 2.97 |
| attack alt | `ExoGame_Troll_Attack` | 1.93 |
| cast | `ExoGame_Troll_Attack_Vomit` | 4.80 |
| roar | `ExoGame_Troll_Aggro` | 2.60 |
| dash | `ExoGame_Troll_Attack_Charge` | 2.60 |

Its skeleton carries 29 clips in all (`Data/Authoring/Souls/body-clips.json`); the table lists the ones the soul names.

Design sheet: `docs/design/souls/ogre.md` in the repository. Mesh `SK_Troll_Poison`.

Pictures are frames from a game rendered off screen on 2026-10-28, the body standing idle in the town (grey-box map), HUD cropped away. Clip lengths are seconds at play rate 1, read from the clip assets (`Data/Authoring/Souls/clips.json` and `body-clips.json` for the souls).
