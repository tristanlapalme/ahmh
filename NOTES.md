# AHM Gatineau stats page — notes

Personal project. Started 2026-11-25 (moved here from another repo's session).

## Goal

One central web page where my son can see all his minor hockey stats.
Very low use (just us), so: no caching, no hidden key, no worry about the API changing.

## Decisions — architecture

- Static site: **one file `site/index.html`, plain HTML/CSS/JS**. No build step, no framework, no server, no proxy.
- Host: **Cloudflare** (chosen over GitHub Pages: no public repo needed). See Deployment below.
- API key goes straight in the JS. If it stops working (422), copy the new one from DevTools on page.spordle.com.
- No caching. Every page view hits the Spordle API live.
- UI language: **French** (labels, positions, roles, infractions via `i18n.fr`). Conversation with Claude is in English.
- Team pages work for **any team** via `?team=<id>`; nothing team-specific is hardcoded except the home page's league (201289) and the highlighted team (214085).
- Office IDs are never hardcoded per request: always use the selected team's own `officeId`.
- Deploy only when explicitly asked.

## Decisions — UI (all confirmed by user on 2026-11-25)

Responsive / phones (added 2026-11-26, breakpoint `max-width: 560px`)
- Every page, tab and modal must work on a phone (~390px) with **no horizontal scrolling**.
- Tabs (page + modal) share the width equally instead of overflowing.
- Tables: team/player names wrap; low-priority columns (`.opt`: %, BP, BC, PUN in standings; PÉN in stats) hidden.
- Horaire rows: date + time on one line, details below, result pill on the right.
- Modals become **full-screen sheets**; title bar fixed, only the body scrolls (`.dlg-body`).
- Desktop: stats modal uses the wider `.wide` dialog (880px), box score 680px.
- How it was checked: Playwright screenshots at 390px and 1100px (installed in the session scratchpad, uses the existing Chromium in `~/.cache/ms-playwright/chromium_headless_shell-1234`).

Navigation
- **Entry point = league standings** (`/`). Not a team page. No tabs on the home page.
- Clicking a team name in any standings table opens that team's page.
- Team page = header (logo, name, "U11 · 2026-27") + **3 pill tabs, matching Spordle: Cahier d'équipe · Horaire · Classement**. Default tab: Cahier d'équipe.
- **No "← Classement" back link** — the Classement tab plays that role.

Cahier d'équipe
- Two tables: Joueurs (sorted by number; #, Nom, Position; C/A badges) and Personnel (Nom, Rôle).
- Names title-cased ("Jérémi Desbiens"), positions/roles translated to French.
- **"Stats" pill in its own last column** (right-aligned, empty header) of the Joueurs table (players only, not staff) → opens the shared modal titled "Statistiques" (requested 2026-11-26).
  - Stats are **computed client-side from the box scores of all played games** (same games call as Horaire; there is no per-player stats endpoint in use). Match on `participantId` + `teamId === TEAM_ID`.
  - Content: big number + name + position; "Totaux 2026-27" table (rows Saison / Tournoi when both exist, then Total; columns B, A, PTS, PÉN, PUN); "Par match" table listing only games where the player has a point or penalty; each row is tappable (score pill + "Détails ›") → box score.
  - **No games-played column**: attendance isn't published, so a GP count would be the team's, which is misleading.
  - PUN: Minor 2, Double Minor 4, Major 5, Misconduct/Game Misconduct 10, unknown 0 (only "Minor" seen so far).
  - Affiliate call-ups who score (e.g. participant 783104) aren't on the roster (`isAffiliate:false`), so they get no pill.

Horaire
- **Games and practices merged in one chronological list**, grouped by month, whole season.
- **No filter toggles / checkboxes / Matchs-Pratiques pills** — tried and rejected by user. Do not re-add.
- Next upcoming event highlighted; past events dimmed.
- Tag colours: **Saison = blue, Tournoi = purple, Hors concours = teal, Pratique = orange** (tournament must differ from league).
- Opponent name links to the opponent's team page. Game `comments` not shown (can be private coach notes).
- Played games (revised 2026-11-26): **the whole row is the tap target** (opens the box score), with the coloured score pill ("V 4–2") and a small **"Détails ›"** hint under it. Same in the stats modal's "Par match" table.
  - Replaces the earlier "clickable score pill + tooltip" decision: people didn't see the pill as clickable, and tooltips don't exist on phones.
  - The opponent link inside the row still navigates to the opponent's page (does not open the modal). Rows are keyboard-accessible (Enter/Space).
  - Past rows dim only the date/details, never the score + "Détails ›".
  Pill colours: win = green, loss = pink/red, tie = grey.

Sommaire du match (modal)
- One shared `<dialog>` (id `boxscore`, title in `#dlg-title`) for both box score and player stats; content is swapped in place when navigating stats → score pill → box score. Closes with ✕, Escape or backdrop click.
- Games list and each box score are fetched once per page load and reused (Horaire, box score, player stats).
- Header: away team left, home team right, big score, date · arena · competition · game #.
- "Pointage par période" table, then **tabs: Buts (n) · Pénalités (n)**, default Buts.
- Goals/penalties grouped by period, sorted, elapsed time shown; AN/DN/FD/TP/CSC badges; assists or "Sans aide".
- **Colour rule: the winner is highlighted in green** (header name + period-table row), same green as the win pill. Never red for a winner. Tie → no highlight.
- Open question (asked, not answered yet): goal/penalty rows of the selected team still have a yellow background — keep, change or remove?

Standings
- API `points` shown as-is (league awards extra points; not 2W+T).
- Selected/highlighted team row: yellow background, team name in red accent. (Not yet revisited after the "winner = green" rule.)

## Source

- Public site: https://page.spordle.com/fr/ahm-gatineau/teams/181953 (that team is Ambassadeurs 1, not ours — just where the API was discovered)
- API base: `https://pub-api.play.spordle.com/api/sp`
- Looks like a LoopBack API: `filter` query param is JSON (`where`, `include`, `limit`, `inq`, `or`...).

### Auth (required)

Header:

```
Authorization: API-Key f08ed9064e3cdc382e6abb305ff543d0150fb52f
```

Without it: `422 Invalid Permissions (custom.unauthorized)`.

### CORS — checked, works

- The API echoes back any `Origin` in `access-control-allow-origin`.
- OPTIONS preflight returns 204, allows GET, max-age 86400.
- Exposed headers: `X-Total-Count`, `X-Client-Version`, `X-Authorized-Roles`.
- So `fetch()` from any domain works with the Authorization header.

```js
const res = await fetch(url, {
  headers: { Authorization: 'API-Key f08ed9064e3cdc382e6abb305ff543d0150fb52f' }
});
const data = await res.json();
```

## Known endpoints

### Teams (tested: 200, 199 teams)

```
GET /teams?filter={"include":["category"],"where":{"seasonId":"2026-27","or":[{"effectiveOffices":6090},{"officeId":{"inq":[6103,8581,11236]}}]}}
```

- Son's team: **FESTIVALS HULL, ID 214085** (U11 BB). (181953 = AMBASSADEURS 1 GATINEAU, not his team.)
- Office IDs: 6090 (main), 6103, 8581, 11236
- Season ID format: `"2026-27"`
- Team fields seen: `id, name, shortName, division (e.g. "U15"), gender, categoryId, seasonId, officeId, logoUrl, homeColor, awayColor, isPublic, category{...}`

### Team's schedules (tested)

```
GET /teams/214085/schedules
```

- 201289 — League: LHRO M11 BB - Saison régulière (2026-10-13 → 2027-02-24)  ← standings page uses this
- 196523 — Tournament U11 BB (2026-11-05 → 11-08), 4 pools (Groupe E/F/G/H)
- 196032 — Tournament U11 BB (2027-01-04 → 01-10)
- 196017 — Tournament U11 BB (2027-01-25 → 01-31)

### Standings (tested)

```
GET /schedules/{scheduleId}/teamStats?filter={"include":["team"],"where":{"team":{"name":{"neq":"TBA"}}}}
```

- One row per team: `ranking, manualRanking, poolId, poolRanking, gamesPlayed, wins, losses, ties, otl, sol, points, pointsPct, goalFor, goalAgainst, diff, pim, ...` + `team{name, logoUrl}`.
- `points` is NOT 2W+T in the league (extra points system) → display the API value as-is.
- Pool names + hidden stats: from `GET /schedules?officeId=6090&seasonId=2026-27` (the `where` filter is ignored; find by id). Each schedule has `pools[{id,name}]` and `settings.hiddenStats`.

### Roster — "Cahier d'équipe" tab (tested)

```
GET /members?filter={"where":{"positions":{"inq":["F","C","D","G","Head Coach","Assistant Coach","Goaltending Coach","Manager","Trainer","Safety Person"]},"isAffiliate":false,"isReleased":false,"teamId":214085}}
```

- `participant{firstName,lastName,fullName,pictureUrl}` comes back without needing `include`.
- Fields: `positions[]` (a person can have several, e.g. Manager + Safety Person), `number`, `isCaptain`, `isAlternate`.
- Names are ALL CAPS in the API; the page title-cases them.

### Schedule — "Horaire" tab (tested) = games ∪ practices

```
GET /games?filter={"order":["startTime ASC","number ASC"],"where":{"and":[{"date":{"between":["2026-08-01","2027-07-31"]}},{"or":[{"awayTeamId":214085},{"homeTeamId":214085}]}]},"include":["teamStats","surface","awayTeam","homeTeam"]}
GET /practices?filter={"where":{"date":{"between":[...]},"venue":{"neq":null},"or":[{"officeId":6090},{"secondaryOfficeId":6090}],"teams":{"id":214085}},"fields":{"recurrence":false},"include":["practiceType","surface","teams"]}
```

- Practices REQUIRE an office filter (`422 officeId or effectiveOffices is required`). Use the **team's own `officeId`** (6090 = AHM HULL, 6089 = AHM GATINEAU). Same results as Spordle's list `[6090,6103,8581,11236]` (those are sibling associations under parent 6087).
- Game: `score` = `{ "<teamId>": goals }`, `teamStats[].gameResult` = win/loss/tie, `scheduleId` → league vs tournament, `surface.venue.name` = arena.
- Practice: `practiceType` can be null; `practiceType.i18n.fr.name` for French; `teams[]` can include other teams sharing the ice.
- Game `comments` can be internal notes between coaches → not displayed.
- Date range = whole season, derived from team `seasonId` ("2026-27" → 2026-08-01..2027-07-31).

### Box score — "Sommaire" modal on played games (tested, league + tournament)

```
GET /games/{gameId}/boxScore
```

- `{ gameNumber, teams[{id,name,logoUrl}], goals[], shootout[], penalties[] }`.
- `goals[]`: `gameTime{period,"minutes/seconds" = clock remaining, elapsedMinutes/elapsedSeconds}`, `teamId`, `participant{fullName,number}`, `assists[]` (same shape), `isPowerplay/isShorthanded/isEmptyNet/isPenaltyShot/isOwnGoal`.
- `penalties[]`: same `gameTime`, `infraction` + `i18n.fr.infraction`, `duration{name, i18n.fr.name}` (e.g. Minor/Mineure), `isBenchPenalty`.
- Items are NOT sorted → sort by period then elapsed time. Page shows elapsed time.
- Unplayed games return empty goals/penalties; the page only links played games (those with `score`).
- `shootout[]` shape not seen yet (always empty so far) — page shows only player names if present.

## App structure

- `site/index.html`, single file.
- Home page `/` = league standings (schedule 201289), FESTIVALS HULL highlighted, each team name links to its team page.
- Team page `/?team=<id>` = 3 tabs (no separate back link — the Classement tab plays that role).
- Tabs (hash routes): `#cahier` (roster), `#horaire` (games + practices in one list; no filter toggles — user didn't want them), `#classement` (league found via `/teams/{id}/schedules` where `type === "League"`).

### Other notes

- `GET /seasons` needs `officeId` (`400 officeId is a required argument`).

## Deployment (done 2026-11-25)

- Live: **https://s.ahmh.workers.dev** (renamed from `ahm-standings` on 2026-11-26 — user didn't like the long name; URL = `<worker name>.<account subdomain>.workers.dev`).
- Old Worker `ahm-standings` (https://ahm-standings.ahmh.workers.dev) still exists until deleted (`npx wrangler delete --name ahm-standings`) — only on user's confirmation.
- Cloudflare "Pages" is now Workers + static assets. Config: `wrangler.jsonc` (name `s`, assets dir `site/`).
- Redeploy: `npx wrangler deploy` from `~/ahmh` (wrangler already logged in).
- A new workers.dev URL gives SSL handshake errors for ~2 min until the cert is issued.

## Status / next steps

- Done & deployed (2026-11-25): standings home page, team pages with the 3 tabs, box score modal.
- Done & deployed (2026-11-26): player "Stats" pill + stats modal in Cahier d'équipe, responsive layout for phones, tappable game rows with "Détails ›".
- Open: yellow highlight on selected team's goal/penalty rows in the modal (see UI decisions).
- Ideas not started: tournament standings by pool (pool names via `/schedules?officeId=…&seasonId=…`).

## Workflow

- Verify changes by extracting the `<script>` and running it in Node with a stubbed DOM against the live API (scripts were kept in the session scratchpad, not in the repo).
- Git repo (since 2026-11-27), personal GitHub only: https://github.com/tristanlapalme/ahmh (never the work GitLab).
  Repo-local identity `Tristan Lapalme <tristanlapalme@gmail.com>` (the machine's global git email is the work one — don't rely on it). The user pushes themselves.
- `.wrangler/` is git-ignored.
