# Tourney Page Plan

## Goal
Build a simple `tourney.html` page that uses Firebase Realtime Database to support Waddies Championship viewing and team intake.

## Scope
- Read-only viewing (schedule, standings) for anyone with tournament ID (or join code) + PIN.
- Owner-only team intake (no scoring on web).
- Match branding/tone of the site.

## Data Model (RTDB paths)
- `/tournaments/{tid}/meta`
  - `name` (string)
  - `ownerUid` (string)
  - `scoringPasswordSalt` (string)
  - `scoringPasswordHash` (hex or base64 of SHA-256(salt + pin))
- `/tournaments/{tid}/teams/{teamId}`
  - `{ name: string, players: [{ id?: string, name: string }, ...] }`
- `/tournaments/{tid}/schedule/{itemId}`
  - `{ round: number, court: string, startTime: number, team1Id: string, team2Id: string, status: 'pending'|'in_progress'|'completed', finalScore?: string }`
- `/tournaments/{tid}/standings/{teamId}`
  - `{ wins: number, losses: number, points: number }`
- Join code mapping (confirm with app):
  - `/tournament-codes/{code} = { tournamentId: '{tid}' }`

## Permissions & Security
- Reads: allowed for `/tournaments/{tid}/(meta|teams|schedule|standings)` once viewer supplies correct PIN (client validates PIN; server rules remain read-permissive under this subtree).
- Writes (owner-only): `/tournaments/{tid}/teams/**` permitted only when `auth.uid == meta.ownerUid`.
- No scoring UI/actions on the web page.
- Auth: Anonymous for viewers; owner signs in (same account as `meta.ownerUid`).
- App Check recommended/enforced for web.

## Join Flow (client)
1. Inputs: join code or tournament ID, and a 4-digit PIN (numeric).
2. If join code provided, resolve `/tournament-codes/{code}` to get `{tid}`.
3. Fetch `/tournaments/{tid}/meta`.
4. Validate PIN: compute `SHA-256(salt + pin)` (Web Crypto API) and compare to `meta.scoringPasswordHash`.
5. On success: store `{tid}` in `localStorage`. Do NOT store the PIN.

## UI Structure
- Header/banner with tournament name and owner badge when applicable.
- Join panel (shown if not joined): Code/ID + PIN, Join button, inline errors.
- Teams intake (owner only): Form with team name and up to 2 player names; list of teams (live).
- Schedule view (read-only): table with Round, Court, Start Time (local), Team1 vs Team2, Status (+ final score if completed).
- Standings view (read-only): table with Team, W-L, Points.

## Realtime Listeners (after join)
- `/tournaments/{tid}/meta` → banner.
- `/tournaments/{tid}/teams` → team list and name resolution for schedule/standings.
- `/tournaments/{tid}/schedule` → schedule table.
- `/tournaments/{tid}/standings` → standings table.

## Auth Handling
- Initialize Firebase (Auth + Database).
- Anonymous sign-in for viewers.
- Provide “Sign in as owner” (email/password) to enable team intake.
- On auth state change, compare `auth.uid` to `meta.ownerUid` to toggle owner UI.

## Local Persistence
- `localStorage.tourneyId = '{tid}'` to auto-join; do not persist PIN.

## Error Handling
- Invalid code/ID: “Tournament not found”.
- Bad PIN: “Invalid PIN”.
- Network errors: retry; degrade gracefully.
- Permission errors on write: “Owner account required”.

## Security Model (Rules Sketch)
Note: Align with existing app rules. Example idea:
```
"tournaments": {
  "$tid": {
    "meta": { ".read": "auth != null" },
    "teams": {
      ".read": "auth != null",
      ".write": "auth != null && auth.uid == root.child('tournaments').child($tid).child('meta').child('ownerUid').val()"
    },
    "schedule": { ".read": "auth != null" },
    "standings": { ".read": "auth != null" }
  }
}
```

## Tech Notes
- SDKs: `firebase-app-compat`, `firebase-auth-compat`, `firebase-database-compat`, `firebase-app-check-compat`.
- Hashing: Web Crypto `crypto.subtle.digest('SHA-256', bytes)`; ensure same encoding as app (hex/base64).
- Time: `new Date(startTime*1000).toLocaleString()`.

## Acceptance
- Owner can add a team on the web and see it appear on the device before matches start.
- Schedule and standings reflect device data in realtime.
- No scoring or submission actions exist on the web page.


