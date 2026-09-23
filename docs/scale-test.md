# Scale test (S-2.6, honest partial)

*Roadmap item: `ROADMAP-SERVER.md` S-2.6. Feeds DS-2.3 only with the caveat below — this is not
a pass/fail and it is not the dedicated-server number.*

## What this is, and what it is not

No dedicated server binary exists yet (`docs/decisions/DS-0.10.md`: the installed engine refuses
Server targets). This page is the honest partial the DS-0.10 memo names for S-2.6: a listen host
plus a handful of real, headless joiners, loaded with server-spawned bots at the player pawn
class so the replicated object count and the combat traffic resemble players, not field monsters.

**A listen host's numbers include its own render thread and its own client.** They are a lower
bound for what a dedicated server (no rendering, no local client) would show at the same player
count, not a measurement of it. Nothing here is a pass/fail against a player cap; DS-2.3 (the
default player cap) stays blocked on a real dedicated-server run per DS-0.10 §2.

## Method

- **Host:** `UnrealEditor.exe SandboxARPG.uproject /Game/Levels/Lvl_Slice?listen -game -log -windowed -nosteam -port=7787`, one Development-build editor-target process, windowed (so `send_console.ps1` could type into it).
- **Joiners:** two headless `UnrealEditor.exe SandboxARPG.uproject 127.0.0.1:7787 -game -log -nullrhi -nosound -unattended` processes on the same machine (method B, `ROADMAP-SERVER` run plan §1). Two, not the plan's three to five: this machine's earlier sessions this week already established the pattern at two: `docs/systems/replication.md`'s two-window checks, and the S-1.5/S-3.2a tests cited in the run plan all used two. Both connected (`Join succeeded` x2 in the host log).
- **Load:** `Slime.SpawnBots <N> DA_Soul_Corpse` (ROADMAP-SERVER S-2.6, this branch) spawns N server-authority `AMonsterCharacter` pawns — the real player pawn class, with its attribute set, soul component, gem component and ability system component, not `AFieldMonster` — each possessed by a new `AScaleBotController` (`AAIController`, `bWantsPlayerState = true`) that wanders to a random nav point and fires its soul's first move on an independent, staggered timer (no per-frame tick, CLAUDE.md 6.6). Run cumulatively in one host process: 50, then +20 to 70, then +30 to 100, each held for 25 to 30 s before the next batch, so the 70 and 100 rows include the 50's and 70's bots respectively (real growth, not a fresh host each time — a fresh host per count was the original plan but was not necessary to get honest incremental numbers, and running one host avoids three separate warm-up periods skewing the low-N rows).
- **Stats:** `Slime.ScaleStats 5` (this branch), a repeating server-side log line every 5 s: bot count, persistent-level actor count, last frame's delta in ms, and per-connection `InBytesPerSecond`/`OutBytesPerSecond` averaged across the two joiner connections (`UNetConnection`, native engine counters, not a custom estimate).
- **Raw log:** every line kept at `C:/Users/jeffr/.claude/scratch/s-2-6/scale-spike-host.log` (not in the repository — a local capture, per CLAUDE.md 7.13 keeping no absolute path in a tracked file).

## Numbers

Each row is the mean of six `ScaleStats` samples (30 s) taken while the host also carried two
connected joiners; `lastFrameMs` is the engine's last-tick delta at each log line, not a windowed
average, so it is noisy frame to frame but stable across the six samples at a given N.

| Bots (N) | Level actors | Mean last-frame ms | Approx. host FPS (1000/ms) | Mean per-connection in (KB/s) | Mean per-connection out (KB/s) | Ensures/crashes |
|---|---|---|---|---|---|---|
| 50 | 187 | 8.65 | ~116 | 4.48 | 18.3 | 0 |
| 70 | 246 | 9.55 | ~105 | 4.40 | 21.2 | 0 |
| 100 | 336 | 9.85 | ~102 | 4.50 | 32.7 | 0 |

No `Ensure condition failed`, `Assertion failed`, or `Fatal error` line at any point in the run
(`grep -c` on the raw log: 0). No navmesh, spawn or GC warnings beyond the three
`RegistrationFailed_DataPendingKill` lines every session on this map logs at startup (a known
gap, `ROADMAP.md` §4, unrelated to bot count). The Corpse-soul stall named in the task ("bots
sometimes stall in `Corpse_alert_idle`") was watched for and not seen: every spawned bot's log
line for that clip reads the normal one-shot idle-clip play (`playing 'Corpse_alert_idle' (1.33
s)`) followed by ordinary move activity, not a repeat or a stuck state, at any of the three
counts.

## Reading the numbers

- **Server frame time** rose from ~8.7 ms to ~9.9 ms mean between 50 and 100 bots — about 14%
  over double the load, on a host that is also rendering its own client and driving a Slate
  window. A dedicated server with no render thread would very likely read faster in absolute
  terms; whether it shows the same *shape* of growth (roughly linear so far) is exactly what
  DS-2.3 needs a real server process to confirm, per the caveat above.
- **Bandwidth per joiner connection** grew from ~18 to ~33 KB/s outbound as bot count went 50 to
  100 — closer to linear-with-N than flat, which is the expected shape before any relevancy or
  prioritisation work narrows what a distant player is told about (S-2.3's filter pass, `S-2.3a`/`S-2.3b`, is already on `main`; this run did not disable it to compare, so these numbers already include that filtering, not a naive broadcast-everything baseline).
- **Iris object headroom.** Corrected 2026-09-23 (this page had it backwards): `Config/DefaultEngine.ini`
  *does* carry a `MaxReplicatedObjectCount` override, set to 16384 on both `ReplicationSystemConfigServer`
  and `ReplicationSystemConfigClient` under `[/Script/OnlineSubsystemUtils.IpNetDriver]` (S-2.3a,
  `docs/systems/replication.md`'s Object pool note, `PROVISIONAL(DS-2.3)`), same as the engine's own
  built-in default — the config line pins that default in the tracked file rather than leaving it
  implicit, it does not raise or lower it. 100 bots, each contributing on the order of half a dozen replicated
  objects (the pawn, its `UMonsterAbilitySystemComponent`, `UMonsterAttributeSet`,
  `USoulComponent`, `UGemEquipmentComponent`, the `AScaleBotController` and its `APlayerState`),
  is on the order of 700 to 800 objects at N=100 — comfortably inside a 16384 ceiling, alongside
  the level's own actors (336 at N=100). This is an estimate from the class list, not a counted
  read of Iris's own object table; no console command in this codebase prints that count today.

## Legacy driver rerun (2026-09-23, `docs/s26-legacy-driver`)

Driver-agnostic check per `docs/systems/replication.md`'s row (C-3 rule: "PIE tests run with
`net.Iris.UseIrisReplication` at both values") and the S-2.6 row's own ask: the same method at
50 and 100 bots with the legacy driver instead of Iris. The cvar was set in the user config
layer (`Saved/Config/WindowsEditor/Engine.ini`, `[ConsoleVariables]` section,
`net.Iris.UseIrisReplication=0`), not on the command line — an `-ExecCmds` argument runs after
the `?listen` URL has already created the net driver on this map, too late to change which
model it picks, while the `[ConsoleVariables]` ini section is read at engine start before any
world loads. The log confirms it took: `LogNet: InitBase GameNetDriver (NetDriverDefinition
GameNetDriver) using replication model Generic`, against `using replication model Iris` on the
default config. One real joiner this time, not two (`UnrealEditor.exe SandboxARPG.uproject
127.0.0.1:7779 -game -nullrhi -nosound -unattended`, `Welcomed by server` in its own log), so
`connections=1` in every `ScaleStats` line below rather than the Iris run's two — the per-
connection averages are not directly comparable row for row against the Iris table above on
that account, only the shape (frame time and per-connection bandwidth growth from 50 to 100).
Bots raised 50 then +50 to 100 in the same host process, six `ScaleStats` samples per level.

| Bots (N) | Level actors | Mean last-frame ms | Approx. host FPS (1000/ms) | Mean per-connection in (KB/s) | Mean per-connection out (KB/s) | Ensures/crashes |
|---|---|---|---|---|---|---|
| 50 | ~181 | 8.47 | ~118 | 4.06 | 23.56 | 0 |
| 100 | 331 | 10.11 | ~99 | 3.82 | 42.68 | 0 |

No `Ensure condition failed`, `Assertion failed` or `Fatal error` line in the run (`grep -ci` on
the raw log: 0). Frame time rose from ~8.5 ms to ~10.1 ms mean, 50 to 100 bots — about 19% over
double the load, close to the Iris run's ~14% over the same range and within the noise of a
single-joiner, one-host comparison; nothing here says one driver is faster than the other, only
that neither falls over or diverges wildly from the Iris shape at this scale. Per-connection
outbound bandwidth is higher in absolute terms than the Iris table (23.6 -> 42.7 KB/s here
against 18.3 -> 32.7 KB/s there), which is expected and not a driver verdict: this run's one
joiner is the only connection the average is taken over, while the Iris run averaged across two,
and Iris's own relevancy/priority machinery (S-2.3a/S-2.3b) is not identical to the legacy
driver's replication graph defaults on this project (no custom Replication Graph, `CLAUDE.md`
6.7 keeps GAS types out of gameplay code but says nothing about the driver's own filtering).
Raw log at `C:/Users/jeffr/.claude/scratch/live-evidence/s26_host.log` (not in the repository,
per CLAUDE.md 7.13).

## What a dedicated-server run must repeat

Per DS-0.10 §2 and the run plan (§1, method note): every number on this page is a listen-host
lower bound. When a real `SandboxARPGServer` binary exists (DS-0.10 §5.2, on the rented build
host), S-2.6 repeats this exact method — `Slime.SpawnBots` and `Slime.ScaleStats` are already
built for it, nothing here is thrown away — with the server process carrying no local client, at
the same three counts and, per the original row, with the real target of three to five joiners.
Only then does DS-2.3 have a number it can set a default player cap from.

## Provisional

None. `Slime.SpawnBots`, `Slime.ScaleStats` and `AScaleBotController` are dev tooling compiled
out of Shipping (CLAUDE.md 7.7-pattern used throughout `Server/*ConsoleCommands.cpp`), not a
game rule, and write nothing persistent (CLAUDE.md 1.9 does not apply — nothing here is a save).
