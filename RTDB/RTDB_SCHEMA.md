# Waddies RTDB Schema Manual

Version: 1.0 (2025-08-10)

This document describes JSON structures stored in Firebase Realtime Database by the Waddies app. All timestamps are UNIX epoch seconds.

## Conventions
- Types: string | number | boolean | object | array | null
- IDs:
  - playerId: email-hash (preferred) or local UUID string when email absent
  - gameId, teamId, ninerId: UUID strings
- schemaVersion: integer, monotonically increasing. Current game schemaVersion: 2

---

## /games/{gameId}
Top-level object representing a single completed game. New fields include `schemaVersion` (v2) and `scoringMode`.

Fields:
- id (string) [v1]
  - The game UUID string
- date (number) [v1]
  - Game start/end date used for ordering; seconds since epoch (may reflect `endTime` when available)
- isComplete (boolean) [v1]
  - Always true for uploaded games
- noStats (boolean) [v1]
  - If true, exclude from statistics
- isDisputed (boolean) [v1]
  - If true, game is flagged as disputed; exclude from stats until resolved
- excludeFromSpeed (boolean) [v1]
  - If true, exclude from speed-related records
- gameType (string: "1v1" | "2v2" | "3v3") [v1]
  - Derived from team sizes
- schemaVersion (number, integer) [v2]
  - Game schema version; currently 2. Used to gate migrations/backfills
- scoringMode (string: "standard" | "teamAttempts") [v2]
  - Indicates Pro Stats (team attempts) if "teamAttempts"
- tournamentId (string, optional) [v1]
  - UUID for the tournament this game belongs to
- isPlayoffGame (boolean, optional) [v1]
  - True for playoff games
- playoffRound (number, optional) [v1]
  - 1: Quarterfinal, 2: Semifinal, 3: Final
- playoffMatchNumber (number, optional) [v1]
  - Match index within the round
- startTime (number, optional) [v1]
  - Game start time (epoch seconds)
- endTime (number, optional) [v1]
  - Game end time (epoch seconds)
- teams (object) [v1]
  - team1 (object)
    - id (string)
    - name (string)
    - color (string) — human-friendly color name
    - players (object) — { playerId: true }
    - score (number)
  - team2 (object)
    - Same shape as team1
- winner (object, optional) [v1]
  - id (string)
  - players (object) — { playerId: true }
- rounds (array<object>) [v1]
  - Each round has:
    - roundNumber (number)
    - timestamp (number)
    - scores (object) — { playerId: number } points assigned to tossers that round
    - team1Score (number)
    - team2Score (number)
    - pointsScored (number) — 0 indicates wash/switch
    - scoringTeam (string) — teamId of scoring team; on wash, set to the hammer holder to enable hammer reconstruction
    - team1TosserId (string, optional) — playerId of team 1 tosser
    - team1TosserName (string, optional)
    - team2TosserId (string, optional)
    - team2TosserName (string, optional)
    - team1AttemptsRaw (object, optional; Pro Stats) — { onBoard: number, holed: number }
    - team2AttemptsRaw (object, optional; Pro Stats) — { onBoard: number, holed: number }
    - waddieStats (object, optional)
      - { playerId: { thrown: boolean, onBoard: boolean, holed: boolean } }
    - closingStats (object, optional; player-rounds computed mirror) — internal use
- updatedAt (server timestamp, set server-side) [v1]
  - Last update time on the server
- lastCorrectionReason (string, optional) [v1]
  - If present, free-text reason for last correction

Notes:
- Hammer/shoot-first are reconstructed from `pointsScored` and `scoringTeam` when 0.
- Pro Stats rounds include teamXAttemptsRaw and per-player booleans; full counts may be provided per-player in future versions.

---

## /players/{playerId}
Canonical player record.

Fields:
- id (string) [v1]
  - Local UUID for the player (immutable once created locally)
- name (string) [v1]
- isActive (boolean) [v1]
- updatedAt (server timestamp) [v1]
- email (string, optional) [v2]
  - Provided by user; drives global playerId hashing
- emailPrivacy (boolean, optional) [v2]
  - Hides email in public lists
- localID (string, optional) [v2]
  - The original local UUID if player later received an email; aids migration
- previousIDs (object, optional) [v2]
  - Map of historical player IDs (email-hash/local UUID) set to true after migrations

Notes:
- Derived stats are not stored here; they are computed from `player-rounds`.

---

## /player-rounds/{playerId}/{gameId}
Per-player view of a game for stats calculations and quick reads.

Children:
- meta (object)
  - gameID (string) [v1]
  - teamName (string) [v1]
  - teamColor (string) [v1]
  - isValid (boolean) [v1]
    - false if the game is excluded/disputed
  - gameType (string: "1v1"|"2v2"|"3v3") [v1]
  - isWin (boolean) [v1]
  - teamScore (number) [v1]
  - opponentScore (number) [v1]
  - teammates (array<string>) — playerIds [v1]
  - opponents (array<string>) — playerIds [v1]
- rounds (object)
  - { roundNumber: { ... } }
  - Each round entry:
    - roundNumber (number) [v1]
    - score (number) [v1]
      - Points credited to this player (tosser on the scoring team) that round
    - timestamp (number) [v1]
    - waddieStats (object) [v1]
      - thrown (boolean)
      - onBoard (boolean)
      - holed (boolean)
    - waddieCounts (object) [v2]
      - thrown (number)
      - onBoard (number)
      - holed (number)
    - closingStats (object, optional) [v2]
      - opportunity (boolean)
      - success (boolean)

Notes:
- `waddieCounts` values are exact when Pro Stats per-player raw attempts exist; otherwise derived heuristically (tosser rounds).

---

## /niners/{ninerId}
User-submitted highlight (9-point round) media.

Fields:
- playerName (string) [v1]
- playerID (string) [v1]
- timestamp (number) [v1]
- description (string) [v1]
- teamName (string) [v1]
- opponentTeamName (string) [v1]
- gameScore (string) [v1]
- location (string, optional) [v1]
- mediaPath (string) [v1]
  - Firebase Storage path to the uploaded media
- isVerified (boolean) [v1]

---

## Backfill and Versioning
- New writes include `games/{gameId}/schemaVersion = 2` and `scoringMode`.
- When a remote game is newer than local (we skip full overwrite), the app backfills missing/outdated fields via a minimal `updateChildValues`:
  - Add `scoringMode` if missing
  - Add or bump `schemaVersion` if missing/outdated

## Validation (RTDB Rules Guidance)
- `games/{id}/scoringMode` must be one of: `standard`, `teamAttempts`.
- `games/{id}/schemaVersion` must be a positive integer.
- `players/{id}` accepts email/privacy fields but denies arbitrary writes to derived stats.
- `player-rounds` are written by the scorer app instance only; consumers should read-only.

## Changelog
- 2025-08-10 (v2): Added `games.schemaVersion`, `games.scoringMode`, `player-rounds.rounds[].waddieCounts`, backfill behavior.
- 2024-.. (v1): Initial schema with games, players, player-rounds, niners.


