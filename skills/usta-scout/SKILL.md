---
name: usta-scout
description: >
  Scouting report skill for USTA league teams. Use this skill whenever the user asks to
  look up, scout, research, or analyze an opponent team in a USTA league — even if they
  just say "look up [team name]", "what do we know about [team]", "prepare for our match
  against [team]", or "pull up [team]'s lineup history". The user will typically provide
  a team name (e.g. "BETC-Downright Smashing-Schiller") and optionally a league format
  (e.g. "18+ 3.0"). Always use this skill for any USTA opponent scouting request. Outputs
  a formatted .docx scouting report with full roster, match-by-match lineup tables, and
  strategy notes.
---

# USTA Opponent Scouting Report

## Overview

This skill scrapes public data from **tennisrecord.com** (no login required) to build a
complete opponent scouting report as a `.docx` file. The report covers:

- Full roster with Dynamic Ratings (DR) and C/S rating type
- Court-by-court lineups from every completed match, both teams shown side by side
- Players who haven't appeared in any match yet (wildcards)
- Strategy recommendations based on lineup patterns

---

## Step 1 — Find the team on tennisrecord.com

Construct the team profile URL:
```
https://www.tennisrecord.com/adult/teamprofile.aspx?year=YEAR&teamname=TEAM-NAME-HERE
```

URL-encode the full team name, not just spaces. Fetch this page and extract:
- Full player roster (names, DRs, season records, singles/doubles splits)
- All match result links — format: `/adult/matchresults.aspx?year=2026&mid=XXXXX`
- The league/format name shown on the page (to confirm it matches the requested league)

**If multiple leagues:** tennisrecord.com uses a `&s=N` parameter to distinguish teams
with the same name in different leagues. If the page shows the wrong league, try `&s=1`,
`&s=2`, etc. until the correct format appears. Ask the user if unsure.

**Year:** Default to the current league year unless the user specifies otherwise.

---

## Step 2 — Fetch all completed match results

From the match links extracted in Step 1, fetch only the ones that have actual scores
(non-zero results). Fetch all completed matches in parallel using WebFetch.

For each match, extract:
- Both team names
- All 5 courts: S1, S2, D1, D2, D3
- For each court: player name(s) on each side, scores, winner

---

## Step 3 — Get C/S rating type for every player

For each unique player who appears in any match (both the scouted team AND opponents),
fetch their profile using the exact `href` found on the team page or match page. Do not
reconstruct the URL from the player name alone, because TennisRecord sometimes needs an
`&s=N` disambiguator for the correct player profile.

Extract the rating designation shown (e.g. `3.0 C` or `3.0 S`). The letter after the
number is the rating type:
- **C** = Computer Rated (USTA has calculated this from match history)
- **S** = Self Rated (player declared their own level — treat as an unknown; they may
  play above their listed rating)
- Any other suffix letter should be treated as **unknown** and displayed as `(—)`, not as
  a first-class rating type with special meaning.

Fetch all profiles in parallel. If a profile returns no data or shows `——`, mark the
player as `(—)`.

Also fetch profiles for all players on the full roster (not just match participants) to
populate the roster table.

---

## Step 4 — Identify players who haven't played yet

Cross-reference every name on the full roster against names that appeared in any match.
Players who never appear in a match are **wildcards** — flag them prominently. They are
unknown quantities who could appear in any court position.

---

## Step 5 — Generate the .docx report

Use the local generator script in this skill directory:

```bash
python3 skills/usta-scout/generate_report.py --team "TEAM NAME" --year YEAR
```

Optional:

```bash
python3 skills/usta-scout/generate_report.py --team "TEAM NAME" --year YEAR --output /path/to/output.docx
```

The script is self-contained and generates the `.docx` directly with `python-docx`.

### Document structure

1. **Title block** — Team name, league, date prepared, season record
2. **Legend** — Explain symbols used (◆, ★, S ⚠, C, color coding)
3. **Full Roster table** — All players sorted by DR descending, columns:
   `# | Player | DR | Rating (C/S) | Season Record | Singles | Doubles | 2026 Status`
4. **One match table per completed match** — Header shows date, opponent, final score.
   Columns: `Court | Scouted Team Player | DR | Result | Opponent Player | DR`
5. **Strategy section** — Court-by-court recommendations based on observed patterns

### Visual conventions

Use these consistently so the report is scannable at a glance:

| Element | Convention |
|---------|-----------|
| DR ≥ 3.0 player | Blue shading + **bold** + ◆ symbol |
| Self-rated player | `(S)` after name, red cell in Rating column, ⚠ symbol |
| Not yet played | Yellow row shading + `NOT YET PLAYED ★` in Status column |
| Scouted team wins a court | Green result cell |
| Scouted team loses a court | Red result cell |
| C/S in match tables | `Player Name (C)` or `Player Name (S)` — always use full name |

### Color values (hex)
- Navy header: `1F3864`
- Light blue (DR 3.0+): `D6E4F0`
- Mid blue (accents): `2E75B6`
- Win green: `E2EFDA`
- Loss red: `FCE4D6`
- Warning yellow (not yet played): `FFF2CC`

---

## Step 6 — Strategy section

Analyze the lineup data and write a strategy table with one row per court (S1, S2, D1,
D2, D3). For each court, note:
- Who the opponent typically puts there (name, DR, C/S)
- Their recent results on that court
- A specific recommendation for the matchup

Also flag:
- Any self-rated players (S) — treat as unknowns regardless of DR
- Players who haven't played yet — could appear anywhere
- Overall team confidence trend (are they winning or losing recently?)

---

## Save location

Save the final `.docx` to the user's workspace folder with a descriptive filename:
`[OpponentTeamName]_Scouting_Report.docx`

The local generator performs basic structural validation before saving. If it raises an
error, treat that as a failed report build and fix the parsing problem before handing the
document to the user.

---

## Tips from prior runs

- **Fetch profiles in parallel** — there can be 15–20+ players; sequential fetching is
  very slow. Batch all profile fetches into a single parallel WebFetch call.
- **tennisrecord.com match IDs ≠ TennisLink match IDs** — the user may give you a
  TennisLink ID; always use tennisrecord.com's own `mid=` parameter from the team
  profile page instead.
- **Use exact profile links** — some players need the page's original `href` with an
  `&s=` suffix; rebuilding the URL from the raw player name can return the wrong person.
- **Full names in match tables** — never abbreviate player names; users need to
  recognize their opponents by full name.
- **Doubles rows** — format as `Player A (C) / Player B (S)` with each player's rating
  type shown individually.
