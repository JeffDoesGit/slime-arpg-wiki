# Builds and the launcher

*Roadmap E-6.6, S-0.12; the rule is DS-0.11 (proposed), the plan `docs/plans/auto-update-2026-10-14.md`. Written 2026-10-14 from `tools/launcher/SlimeLauncher.ps1` and `.github/workflows/package-live.yml` as they stand; nothing of the workflow has run on a runner yet.*

## What it does

Every push to `main` becomes a build, and a collaborator's PC fetches the newest build by itself. Nobody packages by hand before a session and nobody downloads an archive and extracts it. You start the launcher; it checks, updates if there is something newer, and starts the game.

## What you see when you play

- **`SlimeLauncher.cmd`** in a folder of your own. Double-click: a console window says which build is newest, which is installed, downloads if they differ (about 1.7 GB, with curl's own progress line), then starts the game playing alone. `SlimeLauncher.cmd 100.72.164.1` does the same and joins that host.
- **The first start** asks for a GitHub token unless the GitHub CLI is signed in on that PC, in which case it takes the CLI's token and asks nothing. The token is kept encrypted for your Windows account under `%LOCALAPPDATA%\SandboxARPG\Launcher`.
- **The game** always runs with `-SaveToUserDir`: your saves, config and the game's log live under `%LOCALAPPDATA%\SandboxARPG\Saved`, outside the build, so an update never touches them. The game's log is `%LOCALAPPDATA%\SandboxARPG\Launcher\game.log`; the launcher's own is `launcher.log` beside it.
- **An older build back**: `SlimeLauncher.cmd -Pin live/<date>-<sha7>` runs that build and keeps running it until `-Unpin`. One previous build is kept beside the installed one as `Windows.previous`.
- **A refused join**: every live build carries its own version (`0.1.0+<sha7>`), and a client connects only to a host on the same one (DS-5.1). A joiner on last night's build is refused by today's host; starting the launcher brings them level. The wording on screen for that case is E-6.7.

## How a build is made (the workflow)

`package-live.yml` runs on a self-hosted runner that has the engine and the project's LFS content (labels `self-hosted, windows, ue58, package`), on every push to `main`:

1. Tags the merge commit `live/<YYYY-MM-DD>-<sha7>`.
2. Writes `ProjectVersion=0.1.0+<sha7>` into its checkout's `Config/DefaultGame.ini` for the cook and puts the file back after; nothing is committed.
3. Packages the Development Win64 client with the `docs/playtest.md` command (`RunUAT BuildCookRun`, the one map).
4. Smokes it headless: the packaged game runs with `-nullrhi` through a teleport to each well, and a `Fatal error`, `NO PLAYERSTART` or an ensure in its log fails the job before anything is published.
5. Zips the `Windows` folder with `tar.exe`, splits it into numbered parts if it passes 1.9 GB (GitHub caps one asset at 2 GB), writes `manifest.json` (tag, sha, version, size and sha256 of each part, the launcher's version and hash) and zips the launcher itself beside it.
6. Publishes a pre-release on the tag with notes generated from the merged pull requests, then deletes live releases beyond the newest five, tags included.

Three quick merges build once: the job cancels an older run in progress. A merge while no runner is on queues; the next runner online builds the newest `main`.

## How the launcher works

1. Reads the newest release tagged `live/*` through the GitHub API (or the pinned tag). The repository is private, so the request carries the token.
2. Compares the tag with `installed.json` beside the game. Same tag: starts the game.
3. Downloads `manifest.json`, then each part with `curl.exe` (resumable; `--retry 3`), checks each part's sha256 against the manifest (a hand-made release without a manifest is installed with a warning), joins numbered parts by concatenation, extracts with `tar.exe` (a 7-Zip archive from the old hand-made path extracts with 7-Zip when installed).
4. Swaps: the installed `Windows` becomes `Windows.previous`, the new one takes its place; a failed swap puts the previous back. Writes `installed.json`.
5. If the release carries a newer launcher, replaces its own files and starts again with the same arguments.
6. Starts `Windows\SandboxARPG.exe` with the join address if any, `-SaveToUserDir`, `-abslog` to the launcher folder, and whatever `-GameArgs` was given.

## What is settled and what is not

- **Proven** (2026-10-14, Jon's PC): the launcher against `playtest/2026-10-13` as a stand-in release, since no `live/*` release exists: 1.09 GB in 35 s, extracted, installed, the game started and read its record from `%LOCALAPPDATA%`. The GitHub CLI's token picked up with nothing pasted. `-NoUpdate` with nothing installed stops with code 3.
- **Not run**: the workflow itself (no runner carries the `package` label; registering one needs repository admin, which is Jeff's), the smoke step inside a job, a split release, the launcher's self-update, a joiner through the launcher on another PC.
- **Decided** (DS-0.11, Jeff, 2026-10-20): a pre-release per merge to `main`, a launcher per PC, saves in `%LOCALAPPDATA%`; a token is any personal access token that can read this repository, classic or fine-grained, the collaborator's pick (security is deferred to the external-playtest review, `CLAUDE.md` section 12 and `docs/security-deferred.md` SEC-1); the runner is registered by Jeff with the `package` label and stays advisory until the repository is public-safe (CLAUDE.md section 9). The CLAUDE.md 8.2 text is Jeff's own pull request.
- **Not built**: delta updates (a code-only merge still downloads the whole build), a window instead of a console, a pad or Steam path, the dedicated server following the same tag.
