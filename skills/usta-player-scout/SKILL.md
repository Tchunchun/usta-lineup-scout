---
name: usta-player-scout
description: Generate a single-player USTA scouting report (.docx) from public tennisrecord.com data. Use when the user asks to scout, research, or analyze one player by name — e.g. "scout Jane Zhu", "pull up [player] from PNW", "what do we know about [player]", or "analyze [player]'s match history". User provides first and last name, and may also provide an `s` disambiguator or a direct profile URL. Do not use for whole-team lookups (use usta-team-scout) or non-USTA players.
license: Proprietary
compatibility: Requires Python 3.10+ and python-docx for local report writing. All tennisrecord.com collection must use fetch_webpage; the Python renderer must remain offline-only.
metadata:
  version: "2.0.0"
  author: usta-lineup-scout
---

# Per-Player Scouting Report

## Overview

Generate a `.docx` scouting report for one player from public `tennisrecord.com` pages. This skill uses `fetch_webpage` for all network collection, extracts match-history links from the fetched profile page, normalizes the result into JSON, and then runs the local Python renderer with no outbound network access.

## When to use / When not to use

Use when the user asks to scout, look up, or analyze a single player. Trigger phrases: "scout [player]", "pull up [player]", "what do we know about [player]", "analyze [player]'s match history", "how has [player] been playing".

Do not use when:
- The user wants a whole-team report — use `usta-team-scout` instead.
- The player is not on `tennisrecord.com`.
- The user only wants a current rating with no report.

## Inputs

| Input | Required | Default | Notes |
|------|----------|---------|-------|
| player name | yes | — | First and last name as shown on TennisRecord if known. |
| `s` disambiguator | no | none | Append when the user already knows `&s=N` or when the plain profile URL resolves to the wrong player. |
| direct profile URL | no | none | Preferred when the user already has the exact TennisRecord profile link. |
| months | no | `24` | Lookback window for analysis and report output. |
| output filename | no | date-stamped default | File is always written under `reports/`. |

Default output filename: `reports/player_<First>_<Last>_<LocationSlug>_<YYYYMMDD>.docx`.

## Steps

### Step 1 — Resolve the profile URL directly

Do not use the TennisRecord POST search form.

Build the base profile URL directly from the player name:

```text
https://www.tennisrecord.com/adult/profile.aspx?playername=<First>%20<Last>
```

Rules:
- Encode spaces as `%20`, never `+`.
- If the user supplied an `s` disambiguator, append `&s=<N>`.
- If the user supplied a direct profile URL, use that instead of rebuilding it.

Fetch the profile page with `fetch_webpage`.

### Step 2 — Parse the profile page

From the fetched profile page, extract:
- player name
- location
- NTRP level and rating type (`C`, `S`, or unknown)
- estimated dynamic rating and its `as of` date
- the exact match-history links shown on the page

Do not construct match-history URLs independently. Use the links from the profile page as the source of truth. If a fetched `href` contains literal spaces, preserve the link structure and only normalize spaces to `%20` if the fetch tool requires it.

If the rating type is `S`, flag that prominently in the final report.

### Step 3 — Fetch year pages with `fetch_webpage`

Determine which year pages are needed to cover the lookback window, then fetch those exact links from the profile page.

Use one `fetch_webpage` call with all required year URLs when practical.

For page structure and row interpretation, see [references/PARSING_NOTES.md](references/PARSING_NOTES.md).

Normalize each match into this shape:

```json
{
  "date": "2026-03-14",
  "court": "D1",
  "is_singles": false,
  "league": "Adult 18+",
  "level": "3.5",
  "my_team": "Team A",
  "opponent_team": "Team B",
  "result": "W",
  "score": "6-3 6-4",
  "partner": {"name": "Partner Name", "rating": 3.41},
  "opponents": [
    {"name": "Opponent One", "rating": 3.56},
    {"name": "Opponent Two", "rating": 3.48}
  ],
  "match_difficulty": 3.52,
  "dynamic_rating_after": 3.44,
  "rating_trend_hint": "up"
}
```

Deduplicate on `(date, court, sorted opponent names)`.

### Step 4 — Write normalized JSON for the renderer

Before running Python, write a local JSON file containing:

```json
{
  "player": {
    "name": "...",
    "location": "...",
    "ntrp_level": "3.5",
    "rating_type": "C",
    "dynamic_rating": 3.44,
    "rating_as_of": "04/20/2026",
    "profile_url": "https://...",
    "match_history_urls": {"2026": "https://..."}
  },
  "lookback": {
    "months": 24,
    "start": "2024-04-27",
    "end": "2026-04-27"
  },
  "matches": [ ...normalized matches... ]
}
```

The renderer must receive only this local JSON. No network calls are allowed once Python starts.

### Step 5 — Generate the `.docx`

Run the offline renderer from the skill root:

```bash
python3 scripts/player_report.py --input path/to/player_data.json
```

The report includes:
1. Title block with player, location, NTRP, DR, lookback window, and source URL.
2. Self-rated warning when `rating_type == S`.
3. TL;DR bullets.
4. Volume and activity.
5. Results.
6. Opponent strength.
7. Rating trend.
8. Court and partner breakdowns.
9. Notable matches.
10. Full match log.

Visual conventions and formatting: see [references/REPORT_STYLE.md](references/REPORT_STYLE.md).

## Outputs

Final artifact: `reports/player_<First>_<Last>_<LocationSlug>_<YYYYMMDD>.docx`.

Share the generated file with the user once it is written successfully.

## Failure modes

| Signal | Required behavior |
|--------|-------------------|
| Profile page 404s or resolves to the wrong player | Stop and ask the user for an `s` disambiguator or the exact profile URL. Do not guess. |
| `fetch_webpage` cannot access the site, or returns 429 / CAPTCHA | Stop. Suggest trying again later. Do not retry aggressively. |
| Profile page has no match-history links for the needed years | Produce a profile-only report and note that no matches were available in the lookback window. |
| Match-history links include spaces in the query string | Use the extracted link as source of truth. If you must normalize, change spaces to `%20` only; do not rebuild the query from scratch. |
| A numeric field is shown as `-----` or `—` | Normalize to `null` in JSON. Do not coerce to `0`. |
| Rating jumps by more than `0.3` between consecutive matches | Keep the match and flag it in TL;DR and Rating Trend as a possible self-rate bump or data anomaly. |

## Validation

Before handing the `.docx` to the user:
- Confirm the Python command used `--input ...json` and made no network calls.
- Confirm the match log row count equals the filtered match count.
- Confirm every W/L cell is shaded green or red.
- Confirm the report contains no raw HTML, `None`, or `-----` strings.
- Confirm the filename is date-stamped and lives under `reports/`.

If the renderer raises an error, fix the normalized JSON or parser assumptions before delivering the report.
