# Persistence

*Built in roadmap item [S-2.4](../../ROADMAP-SERVER.md), per [`docs/decisions/DS-2.1.md`](../decisions/DS-2.1.md) (what a record holds, atomic write, autosave, backups) and [`docs/decisions/DS-5.1.md`](../decisions/DS-5.1.md) (version stamp, forward-only migration, fixture test). Design rules: GDD 7.5, 7.8, 7.9, 9.10.*

## What it is

`UPersistenceSubsystem`, a `UGameInstanceSubsystem` in
`Source/SandboxARPG/Server/Persistence/`, saves and loads one `UCharacterRecord` per player,
keyed by the validated Steam ID (contract C-7) when one exists, or a stable per-machine dev key
otherwise. It never runs on a client: every read and write happens on the authority, on
`PostLogin`, `Logout`, the autosave timer, and whenever a component that owns saved state fires
`CharacterPersistenceEvents::OnDirty()`.

## The key

`FPersistenceKey::Resolve(const APlayerController*)` asks `FSteamIdentity::GetValidatedNetId` for
a validated Steam ID first. When `IsValidated()` is false (Steam auth is off by default,
`docs/systems/steam-identity.md`), it falls back to a sanitized per-machine name
(`Controller->GetName()`'s host, cleaned to `[A-Za-z0-9_-]`) so a dev session without Steam still
gets a stable, reproducible file. `bIsValidatedSteamId` on the resolved key records which path was
taken, and every save/load log line names it, so a dev-key record is never mistaken for a real
one.

## What is saved

Per DS-2.1 §5, one `UCharacterRecord` per character holds:

- **Progression** — level and current-level XP (`UProgressionComponent::RestoreState`).
- **Souls** — the primary, secondary and worn slot names, and the full unlock list (every soul
  ever granted, whether currently slotted or sitting in the bag — see "the Unlocks bug" below),
  restored through `USoulComponent::GrantSoul` then `EquipSoulByName` /
  `EquipSecondarySoulByName` / `EquipWornSoulByName`.
- **Fragments** — per-monster-type fragment counts (`USoulComponent::RestoreFragments`).
- **Skill tree** — every soul's allocated nodes (`USoulTreeComponent::RestoreAllocations`, backed
  by a new `GetAllAllocations()` getter).
- **Gems** — the bag and every equipped slot (`UGemEquipmentComponent::RestoreGem`).
- **Checkpoint** — the last zone entrance the pawn used, for respawn.

Anything not in that list is not saved, because DS-2.1 does not name it: attack feel and movement
tuning cvars, cooldown/mana state (session-only), party or trade state (unbuilt), and anything an
`OPEN` GDD rule would need to define (e.g. a co-op shared-save question is D-7.1, still open, out
of scope here).

Account-level state (`UAccountRecord`: first-seen and last-seen timestamps, the character list) is
written alongside the character record on login, ahead of D-4.3's per-character-vs-per-account
question — today it tracks one character slot per key.

## Write path

`RecordFileIO::AtomicWrite` writes the new content to `<file>.tmp` **first**. Only once that
succeeds does it roll backups (`.bak1` newest through `.bakN` oldest, the file about to be
replaced becoming the new `.bak1`) and rename the temp file over the live one. A failed temp write
returns `false` having touched neither the live file nor any backup — proven by
`SandboxARPG.Server.Records.FailedWriteLeavesPriorUntouched`, which provokes the failure with a
directory at the temp path and confirms the live file still reads its prior content.

`AutosaveIntervalSeconds` (default 300, 30–3600) and `AutosaveBackupCount` (default 5, 0–50) are
`USandboxServerRules` fields (Config=Game), validated the same way every other server rule is.

## When a record is written

- **Autosave**, every `AutosaveIntervalSeconds`, on a `FTimerHandle` (never a tick).
- **On dirty**, immediately, when `CharacterPersistenceEvents::OnDirty().Broadcast(Owner)` fires —
  bound at every existing `OnSoulSlotsChanged`, `OnGemsChanged` and `OnTreeChanged` broadcast site,
  and at a player pawn's death (`AMonsterCharacter::HandleHealthDepleted`, guarded by
  `IsPlayerControlled()` so a field monster's own death never touches a character record).
- **On logout** (`AMonsterGameMode::Logout`, before `Super::Logout` tears the connection down).
- **On engine shutdown** (`FCoreDelegates::OnEnginePreExit`), a backstop for a clean process exit
  that isn't a per-player logout.
- **On demand**, `Slime.SaveAll` (compiled out of Shipping).

`CharacterPersistenceEvents::OnDirty()` is a plain (non-dynamic) `TMulticastDelegate<void(AActor*)>`
rather than one of the components' existing `DECLARE_DYNAMIC_MULTICAST_DELEGATE` types, because a
dynamic delegate's `AddDynamic(Object, FuncName)` carries no per-broadcast payload — it cannot say
*which* pawn just changed. The subsystem needs that to find the right tracked entry.

## Load path

`PostLogin` calls `PrepareForLogin` **before** `Super::PostLogin` (pawn spawn happens inside the
engine's own `Super::PostLogin`, so the record must already be loaded into a pending slot by
then). `HandleStartingNewPlayer_Implementation` calls `ApplyToPawn` right after
`Super::HandleStartingNewPlayer_Implementation` returns, once the pawn exists. A key with no file
yet gets a fresh, empty record and is logged as such — there is no error path for "no record",
only for a record this build cannot read.

### The Unlocks bug (fixed before merge)

`USoulComponent::GetOwnedSouls()` returns the **bag** only: `EquipSoulByName` moves a soul out of
the bag into `EquippedSoulName` (the E-2.13 "souls are bag items" model this component still
uses). `WriteRecord`'s first cut built `Souls.Unlocks` from `GetOwnedSouls()` alone, so a soul that
was granted and then equipped — the common case — dropped out of the saved unlock list entirely.
On restore, an empty `Unlocks` meant `GrantSoul` was never called for it, so the later
`EquipSoulByName(Record->Souls.Primary)` call refused with `not held`. Fixed by unioning the bag
with whatever is currently in the primary, secondary and worn slots before saving. Proven live: a
joiner granted and equipped a soul, the host process was killed and relaunched, the joiner
reconnected, and the host log showed `SoulComponent: MonsterCharacter_1 body -> DA_Soul_Corpse
(primary)` with no `not held` refusal.

## Versioning and migration

`FSandboxRecordVersion` is a custom version (`FCustomVersionRegistration`) stamped into every
saved record. `IsVersionSupported(int32)` — the one predicate both the real load path and the
automation test use — refuses a record whose version is newer than `LatestVersion`
(`SandboxARPG.Server.Records.VersionMismatchRefused`) rather than guessing at an unknown shape.
Migration is forward-only per DS-5.1: an older record is upgraded as it loads, never downgraded.
`Source/SandboxARPG/Server/Tests/Fixtures/Char_v1.sav` is a committed fixture (a pre-generated,
non-LFS binary `.sav` — no `.gitattributes` pattern currently covers `.sav`, and none was added, so
this file rides in the plain repository) that `SandboxARPG.Server.Records.Migrate` loads and
checks against, proving the migration path against a fixed byte-for-byte input rather than only
against records this session happened to write.

## What DS-2.1 leaves open (not built here)

- The `Local/` single-player store DS-2.1 mentions as distinct from the Steam-keyed store is not
  separated out; a dev-key record and a future local-only record would collide under this scheme.
  No `E-` row exists for it yet.
- Only one character slot per key is restored; DS-2.1's account-vs-character question (D-4.3) is
  answered for souls (unlocks are account-wide per the recorded decision) but the record format
  itself does not yet branch on multiple character slots.
- Session-only state (cooldowns, buffs, current zone beyond the last-used checkpoint) is
  deliberately not saved — per `CLAUDE.md` 1.9, nothing provisional writes persistent data, and
  none of this work rests on a `PROVISIONAL` marker for that reason: every field saved traces to a
  `DECIDED` rule in DS-2.1 or an already-`DECIDED` GDD rule.

## Tests

Five headless automation tests under `SandboxARPG.Server.Records.*`: `RoundTrip`, `BackupRotation`,
`FailedWriteLeavesPriorUntouched`, `VersionMismatchRefused`, `Migrate`. All green alongside the
full `SandboxARPG` suite (92/92, 0 failures) after the Unlocks fix and rebuild.
