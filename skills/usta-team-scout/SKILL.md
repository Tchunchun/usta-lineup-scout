---
name: usta-team-scout
description: Generate a USTA opponent team scouting report (.docx) from public tennisrecord.com data. Use when the user asks to scout, look up, research, or analyze an opponent USTA team — e.g. "scout BETC-Downright Smashing", "look up [team]", "prepare for our match against [team]", "pull up [team]'s lineup history". User provides a team name and optionally a league format (e.g. "18+ 3.0"). Do not use for single-player lookups (use usta-player-scout) or non-USTA leagues.
license: Proprietary
compatibility: Requires Python 3.10+, python-docx, beautifulsoup4, lxml, and network access to tennisrecord.com.
metadata:
  version: "1.1.0"
  author: usta-lineup-scout
---

# USTA Opponent Scouting Report

## Overview

Generate a `.docx` opponent scouting report from public `tennisrecord.com` data. The report includes the full roster, completed-match lineup tables, wildcard players, prior-season wildcard history when available, and a 3-column strategy section with likely lineup predictions and court-level analysis. The generator can also merge manually entered recent-match results from a JSON file when TennisRecord is behind.

## When to use / When not to use

Use when the user asks to scout, look up, or prepare for a USTA opponent **team**. Trigger phrases: "scout [team]", "look up [team]", "prepare for our match against [team]", "pull up [team]'s lineup history", "what do we know about [team]".

Do not use when:
- The user wants a single-player report — use `usta-player-scout` instead.
- The league is not USTA — the data source does not cover it.
- The user only wants to book a court — use `seattle-tennis-booking`.

## Inputs

| Flag | Required | Default | Notes |
|------|----------|---------|-------|
| `--team` | yes | — | Full team name as it appears on tennisrecord.com. |
| `--year` | no | current league year | Four-digit year, e.g. `2026`. |
| `--s` | no | none | `&s=N` disambiguator when multiple teams share a name. |
| `--output` | no | date-stamped default | Filename only; file is always written under `reports/`. |
| `--manual-matches` | no | none | Path to a JSON file of manually entered match results to merge into the generated report. |

Default output filename: `reports/<YEAR>_<Level>_<TeamSlug>_<YYYYMMDD>.docx`.

## Steps

### Step 1 — Resolve the team page

Run the local generator from the skill root's `scripts/` directory:

```bash
python3 scripts/generate_report.py --team "<TEAM NAME>" --year <YEAR>
```

The generator resolves the team profile URL:

```
https://www.tennisrecord.com/adult/teamprofile.aspx?year={YEAR}&teamname={URL_ENCODED_TEAM_NAME}
```

It extracts the full roster (names, DRs, season record, singles split, doubles split, local record), match result links, and the league label shown on the page.

If the page shows the wrong league, append `&s=1`, `&s=2`, … until the correct league appears. Ask the user if unsure.

### Step 2 — Fetch completed matches and player profiles

The generator fetches only matches with actual scores. For each match it extracts both teams, all five courts (S1, S2, D1, D2, D3), player names on each side, scores, and the result from the scouted team's perspective.

It also fetches player profiles in parallel for:
- every roster player
- every participant who appeared in a completed match

From those profiles it records the rating designation (`C`, `S`, or `—`) and display rating string. `S` players must be flagged because they may play above listed level.

### Step 3 — Identify wildcards and prior history

Cross-reference roster names against names that appeared in any completed match. Roster players with no match appearance are wildcards.

For wildcard players with a profile link, the generator attempts to fetch prior-season match history and adds a compact status summary such as `2025: 8W-3L, D2/D3` to the roster table.

### Step 4 — Optionally merge manual recent matches

If the user has match results that are not yet on TennisRecord, pass a JSON file via `--manual-matches`:

```bash
python3 scripts/generate_report.py \
  --team "<TEAM NAME>" \
  --year <YEAR> \
  --manual-matches path/to/manual_matches.json
```

Manual matches are appended to the completed-match sample, included in the match tables, and counted into each roster player's season/local records before the report is written.

Manual JSON schema and an example are documented in [references/MANUAL_MATCHES.md](references/MANUAL_MATCHES.md).

### Step 5 — Build the `.docx`

The generator writes a report under `reports/` and prints:
- the output path
- `completed_matches=<N>`
- `wildcards=<N>`

Document structure:
1. Title block (team, league, date prepared, season record, most recent match).
2. Legend — symbols and color coding.
3. Full roster table — sorted by DR descending.
4. Completed match tables — one per match.
5. Strategy Notes — overall patterns plus a 3-column lineup prediction table.

Visual conventions: see [references/REPORT_STYLE.md](references/REPORT_STYLE.md).

### Step 6 — Strategy Notes

The strategy section has two parts:

1. **Overall Patterns** bullets summarizing:
- dual-match record
- singles record
- doubles record
- self-rated roster players
- wildcards
- DR 3.0+ depth

2. **3-column strategy table** with one row per court (`S1`, `S2`, `D1`, `D2`, `D3`):
- `Court`
- `<Team> Likely Lineup`
- `Analysis`

Each row predicts the most common lineup seen on that court and summarizes prior results on that court. If there is no sample, the row must say `No data yet` / `No completed sample for this court.`

## Outputs

Final artifact: `reports/{YEAR}_{Level}_{TeamSlug}_{YYYYMMDD}.docx`.

Success looks like:
- the file exists under `reports/`
- the roster table renders with at least one row
- the match section either contains one table per completed match or an explicit "No completed matches yet" note
- the strategy section contains all five courts

## Failure modes

| Signal | Required behavior |
|--------|-------------------|
| Team URL returns 404 | Retry fuzzy team-name variants and `&s=1..3`; if still 404, ask the user to confirm the exact team name and year. |
| Page returns CAPTCHA / 403 / 429 | Stop, do not retry aggressively. Report to the user and suggest trying again later. |
| Match page shows a forfeit | Include the court with a `FORFEIT` marker; do not fabricate opponent names. |
| Profile page missing DR | Record DR as `None`, rating type as `(—)`. Do not substitute a default. |
| Roster name has no match-page match | Treat as a wildcard. |
| Multi-league team needs `&s=N` | See Step 1. If ambiguous, ask the user. |
| Zero completed matches | Produce a roster-only report with the note `No completed matches yet — every roster player is currently treated as a wildcard.` |
| `--manual-matches` path does not exist | Fail fast with a clear error message. Do not write a partial report. |
| Manual JSON is malformed or missing fields | Fail fast with a clear error message identifying the missing field or schema problem. |

For HTML-parsing quirks and URL/name-disambiguation edge cases, see [references/PARSING_NOTES.md](references/PARSING_NOTES.md).

## Manual Match Ingestion

TennisRecord often lags by several days after a match is played. When the user has reliable recent results, convert them into the manual-match JSON schema and pass them via `--manual-matches` during report generation.

Trigger phrases: "add this recent match", "merge these results into the report", "TennisRecord is behind", "include this match from last weekend".

If the user starts from a screenshot, transcribe the screenshot into the JSON schema documented in [references/MANUAL_MATCHES.md](references/MANUAL_MATCHES.md) before invoking the script.

The generator will:
- fetch missing opponent DRs and rating types in parallel
- append the manual matches to the completed-match set
- update season/local roster records for the scouted team's players
- include the manual matches in strategy calculations

## Validation

- Confirm the roster table has ≥1 row.
- Confirm every completed-match table has both teams and all five courts listed, or the report contains the explicit no-matches note.
- Confirm no player name renders as a raw URL fragment or `None`.
- Confirm the filename is date-stamped and lives under `reports/`.
- Confirm the strategy table lists `S1`, `S2`, `D1`, `D2`, and `D3`.

If the generator raises an error, treat it as a failed build and fix the parsing issue before delivering.
