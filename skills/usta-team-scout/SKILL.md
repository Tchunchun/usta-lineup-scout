---
name: usta-team-scout
description: Generate a USTA opponent team scouting report (.docx) from public tennisrecord.com data. Use when the user asks to scout, look up, research, or analyze an opponent USTA team — e.g. "scout BETC-Downright Smashing", "look up [team]", "prepare for our match against [team]", "pull up [team]'s lineup history". User provides a team name and optionally a league format (e.g. "18+ 3.0"). Do not use for single-player lookups (use player-scout) or non-USTA leagues.
license: Proprietary
compatibility: Requires Python 3.10+, python-docx, beautifulsoup4, lxml, and network access to tennisrecord.com.
metadata:
  version: "1.0.0"
  author: usta-lineup-scout
---

# USTA Opponent Scouting Report

## Overview

Scrapes public data from `tennisrecord.com` (no login) and produces a `.docx` scouting report covering the opponent team's roster, every completed match's court-by-court lineup, wildcards (roster players who haven't played yet), and court-by-court strategy notes.

## When to use / When not to use

Use when the user asks to scout, look up, or prepare for a USTA opponent **team**. Trigger phrases: "scout [team]", "look up [team]", "prepare for our match against [team]", "pull up [team]'s lineup history", "what do we know about [team]".

Do not use when:
- The user wants a single-player report — use `player-scout` instead.
- The league is not USTA — the data source does not cover it.
- The user only wants to book a court — use `seattle-tennis-booking`.

## Inputs

| Flag | Required | Default | Notes |
|------|----------|---------|-------|
| `--team` | yes | — | Full team name as it appears on tennisrecord.com. |
| `--year` | no | current league year | Four-digit year, e.g. `2026`. |
| `--s` | no | none | `&s=N` disambiguator when multiple teams share a name. |
| `--output` | no | date-stamped default | Filename only; file is always written under `reports/`. |

Default output filename: `reports/<YEAR>_<Level>_<TeamSlug>_<YYYYMMDD>.docx`.

## Steps

### Step 1 — Find the team

Construct the team profile URL:
```
https://www.tennisrecord.com/adult/teamprofile.aspx?year={YEAR}&teamname={URL_ENCODED_TEAM_NAME}
```

URL-encode the full team name. Fetch the page and extract the full roster (names, DRs, season records, singles/doubles splits), all match result links (`/adult/matchresults.aspx?year={YEAR}&mid=XXXXX`), and the league/format name shown on the page.

If the page shows the wrong league, append `&s=1`, `&s=2`, … until the correct league appears. Ask the user if unsure.

### Step 2 — Fetch completed matches

From Step 1's match links, fetch only matches with actual scores. Issue all fetches in a single parallel WebFetch call. For each match extract both team names, all five courts (S1, S2, D1, D2, D3), player names on each side, scores, and the winner.

### Step 3 — Get C/S rating type for every player

For each unique player who appears in any match (both teams) and every player on the full roster, fetch their profile using the **exact `href`** from the team or match page. Do not rebuild the URL from the name — the `&s=N` suffix is load-bearing.

Extract the rating designation (e.g. `3.0 C`, `3.0 S`). `C` = Computer Rated, `S` = Self Rated (treat as unknown; may play above listed rating). Any other suffix or missing value → display as `(—)`.

Issue all profile fetches in a single parallel WebFetch call.

### Step 4 — Identify wildcards

Cross-reference roster names against names that appeared in any match. Match on `href` first (most reliable); fall back to case- and diacritic-normalized name comparison. Roster players with no match appearance are **wildcards** — flag prominently.

### Step 5 — Generate the .docx report

Run the local generator:

```bash
python3 scripts/generate_report.py --team "<TEAM NAME>" --year <YEAR>
```

Optional: `--output <filename>`. The file is always written under the repo-root `reports/` folder regardless of `--output`.

Document structure:
1. Title block (team, league, date prepared, season record).
2. Legend — explains symbols and color coding.
3. Full roster table — sorted by DR descending.
4. One match table per completed match — both teams side by side.
5. Strategy section — court-by-court notes.

Visual conventions, color palette, and doubles-row formatting: see [references/REPORT_STYLE.md](references/REPORT_STYLE.md).

### Step 6 — Strategy section

Write one row per court (S1, S2, D1, D2, D3). For each court note: who the opponent typically puts there (name, DR, C/S), their recent results on that court, and a specific recommendation.

Also flag: self-rated players (S), wildcards, and the opponent's overall recent win/loss trend.

## Outputs

Final artifact: `reports/{YEAR}_{Level}_{TeamSlug}_{YYYYMMDD}.docx` (always date-stamped — roster and match history change over time).

Share the report with the user via a `computer://` link once written.

## Failure modes

| Signal | Required behavior |
|--------|-------------------|
| Team URL returns 404 | Retry with `&s=1..3`; if still 404, ask the user to confirm the exact team name and year. |
| Page returns CAPTCHA / 403 / 429 | Stop, do not retry aggressively. Report to the user and suggest trying again later. |
| Match page shows a forfeit | Include the court with a `FORFEIT` marker; do not fabricate opponent names. |
| Profile page missing DR | Record DR as `None`, rating type as `(—)`. Do not substitute a default. |
| Roster name has no match-page match even after normalization | Treat as a wildcard. Note the ambiguity in the report's footer. |
| Multi-league team needs `&s=N` | See Step 1. If ambiguous, ask the user. |
| Zero completed matches | Produce a report with roster only; note "No completed matches yet — every player is a wildcard." |

For HTML-parsing quirks and URL/name-disambiguation edge cases, see [references/PARSING_NOTES.md](references/PARSING_NOTES.md).

## Manual Match Injection (Screenshot Workflow)

TennisRecord often lags by several days after a match is played. When the user provides a screenshot of recent match results, inject the match directly into the existing `.docx` report rather than waiting for TennisRecord to update.

**Trigger phrases:** "here are the results from [date]", "here's the match screenshot", "add this match to the report", "update the report with [date] results."

### Step-by-step protocol

**1. Parse the screenshot**

Read the screenshot visually. Extract:
- Match date
- Home team name and visiting team name
- For each of the 5 courts (S1, S2, D1, D2, D3):
  - Home player name(s)
  - Visiting player name(s)
  - Set scores and winner

Scores use **match-winner-first convention** — see [Score display convention](#score-display-convention) in `references/PARSING_NOTES.md`.

**2. Look up opponent player DRs and rating types**

For every opponent player who appears in the match (players NOT on the scouted team), fetch their TennisRecord profile:
1. Search `https://www.tennisrecord.com/adult/profile.aspx?playername=<URL-encoded-name>`
2. Extract DR (4-decimal float) and rating type (C/S/—)
3. If the name returns multiple results or the wrong area, try `&s=1`, `&s=2`
4. Issue all fetches in parallel

Do not skip this step — opponent DRs are critical context for strategy notes.

**3. Determine which players are on the scouted team**

The scouted team may be home or visiting. Identify which column belongs to them from the team name headers in the screenshot. Scouted-team players go in the left column of the match table; opponent players go in the right column (consistent with other match tables in the report).

**4. Build the new match table**

Use the same python-docx table structure as the existing match tables in the report:
- Header row: `Court | [Scouted Team] Player | DR | Result | Opponent Player | DR`
- One row per court (S1, S2; then D1, D2, D3 as merged-pair rows)
- Color the Result cell: green for scouted-team win, red for scouted-team loss
- For each court, record the match winner's score first (e.g. `6-3 6-2`), consistent with TennisRecord convention

**Scouted team DRs:** the screenshot does not include DRs. Always pull them from the roster table already in the report (table index 0). Build a `{name: DR}` lookup from the roster and populate the scouted-team DR column from that. Do not leave them blank.

**5. Insert the table into the .docx**

Insert the new match table immediately before the "Strategy Notes" heading. Use lxml `element.addprevious()` to insert at the correct position — do NOT append to the end of the document.

To find the insertion point, iterate document paragraphs and locate the one whose first `<w:t>` text equals `"Strategy Notes"`. Insert before that element's parent `<w:p>`.

Also insert a bolded paragraph above the table with the match date and opponent name, e.g. `"Match 3 — vs FC-Sharma — 4/19/2026"`.

**6. Update wildcards**

For any scouted-team player who appears in the new match for the first time, remove them from the wildcard list in the roster table (change their wildcard marker to their actual DR). Update the wildcard count in the title block if present.

**7. Update the title record**

Increment the W or L in the title block based on whether the scouted team won or lost this match.

**8. Refresh strategy notes**

After inserting the new match, update the Strategy Notes table to reflect the new sample:
- Update court-by-court W-L records to include the new match
- Revise any recommendation that changes given the new result
- Keep strategy notes in the opponent-intelligence frame: *how does this team build their lineup and what should the user watch out for?* (See Step 6 — Strategy section above.)

---
- Confirm the roster table has ≥1 row.
- Confirm every match table has both teams and all five courts listed (forfeits marked).
- Confirm no player name renders as a raw URL fragment or `None`.
- Confirm the filename is date-stamped and lives under `reports/`.

If the generator raises an error, treat it as a failed build and fix the parsing issue before delivering.
