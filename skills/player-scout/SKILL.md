---
name: player-scout
description: Generate a single-player USTA scouting report (.docx) from public tennisrecord.com data, covering recent form, opponent-strength breakdown, singles/doubles split, partner history, and dynamic-rating trend. Use when the user asks to scout, research, or analyze a single player by name — e.g. "scout Jane Zhu", "pull up [player] from PNW", "what do we know about [player]", "analyze [player]'s match history". User provides first + last name and optionally a location. Do not use for whole-team lookups (use usta-team-scout) or non-USTA players.
license: Proprietary
compatibility: Requires Python 3.10+, python-docx, beautifulsoup4, lxml, and network access to tennisrecord.com.
metadata:
  version: "1.0.0"
  author: usta-lineup-scout
---

# Per-Player Scouting Report

## Overview

Scrapes public data from `tennisrecord.com` (no login) and produces a `.docx` scouting report for a single opponent. Complements `usta-team-scout` (which covers whole teams) by zooming in on one player: how they play, who they beat, how their rating is trending.

## When to use / When not to use

Use when the user asks to scout, look up, or analyze a **single player**. Trigger phrases: "scout [player]", "pull up [player]", "what do we know about [player]", "analyze [player]'s match history", "how has [player] been playing".

Do not use when:
- The user wants a whole-team report — use `usta-team-scout` instead.
- The player is not on tennisrecord.com — the data source doesn't cover them.
- The user only wants a player's current rating without analysis — a single search suffices.

## Inputs

| Flag | Required | Default | Notes |
|------|----------|---------|-------|
| `--first` | yes | — | First name. |
| `--last` | yes | — | Last name. |
| `--location` | no | none | Substring match on the search-result "Location" column for auto-pick. |
| `--s` | no | none | `&s=N` disambiguator if already known. |
| `--months` | no | `24` | Lookback window in months. |
| `--output` | no | date-stamped default | Filename only; file is always written under `reports/`. |

Default output filename: `reports/player_<First>_<Last>_<LocationSlug>_<YYYYMMDD>.docx`.

## Steps

### Step 1 — Find the player

POST to the search endpoint:
```
POST https://www.tennisrecord.com/adult/search.aspx
form: firstname=<FirstName>&lastname=<LastName>
```

Results table columns: `Player Name | Location | Gender | NTRP | Updated`. Each row links to `/adult/profile.aspx?playername=<Name>[&s=N]`.

Disambiguation:
- Exactly one result → use it.
- Multiple results → ask the user via `AskUserQuestion`, one option per row (show name + location + NTRP). Do not guess.
- User supplied `--location` and exactly one row matches (case-insensitive substring on the Location column) → use that row and mention which was picked.

Always preserve the exact `href` from the search page — the `&s=N` suffix is load-bearing.

### Step 2 — Pull the profile

Fetch the profile URL. Extract current NTRP + rating type (`3.5 C`, `3.5 S`), Estimated Dynamic Rating (4 decimals) with its "as of" date, Projected Year End Rating (may be `—`), per-year summary rows, and per-year match-history links.

Keep only the years needed to cover the lookback window (`--months`). Example: if `--months=24` and today is `{CURRENT_YEAR}-04-17`, fetch `{CURRENT_YEAR}`, `{CURRENT_YEAR}-1`, and `{CURRENT_YEAR}-2` year pages (the oldest page is date-filtered during analysis).

If the rating type ends in `S`, the player may play above their listed level — flag this prominently in the final report.

### Step 3 — Pull match history

For each target year fetch:
```
https://www.tennisrecord.com/adult/matchhistory.aspx?year={YEAR}&playername=<Name>&s=N&mt=0&lt=0&yr=0
```

Issue year-page fetches in parallel via a single WebFetch call. For per-match table structure, rating-color conventions, and doubles-row parsing, see [references/PARSING_NOTES.md](references/PARSING_NOTES.md).

Parse every match into a structured record with: `date`, `court`, `is_singles`, `league`, `level`, `my_team`, `opponent_team`, `result`, `score`, `partner_name`, `partner_rating`, `opponents[]`, `match_difficulty`, `dynamic_rating_after`.

Deduplicate on `(date, court, sorted opponent names)` to handle matches that appear on year-boundary pages.

### Step 4 — Compute the analysis

Filter matches to the last `--months` months. Compute:

- **Volume & split:** total, singles count, doubles count, matches by level, months-active, longest gap between matches.
- **Results:** overall W-L and win %, singles W-L, doubles W-L, last-10 result string, current streak.
- **Opponent strength:** mean/median/min/max `match_difficulty`, 0.25-bucket histogram, split record vs tougher (+0.25) / peer (±0.25) / easier (-0.25) opponents, notable wins against strong opponents.
- **Rating trend:** first vs last `dynamic_rating_after` in window, max/min/current DR, slope label (rising if slope > 0.002/match, falling if < -0.002, else steady).
- **Position & partners:** court counts (S1/S2/D1/D2/D3), most-common doubles partners with win rate, partner rating range.
- **Notable matches:** top 3 wins (highest `match_difficulty` with W), top 3 losses (lowest `match_difficulty` with L), any bagels / breadsticks delivered or received.

### Step 5 — Generate the .docx report

Run the local generator:

```bash
python3 scripts/player_report.py --first "<FIRST>" --last "<LAST>" --location "<CITY, STATE>"
```

Document structure:
1. Title block (player, location, NTRP + type, DR with "as of" date, lookback window, source URL).
2. Self-rated warning (only if rating type is `S`) — red callout.
3. TL;DR — 3–5 bullets.
4. Volume & Activity.
5. Results (Overall / Singles / Doubles + last-N form + streak).
6. Opponent Strength (difficulty stats + split record + distribution).
7. Rating Trend (first/last/max/min + slope label).
8. Court & Partners.
9. Notable Matches (top 3 wins + top 3 upsets).
10. Full Match Log (every match in window; W/L cell shaded).
11. Footer — scouted timestamp + source attribution.

Visual conventions, color palette, and match-log formatting: see [references/REPORT_STYLE.md](references/REPORT_STYLE.md).

## Outputs

Final artifact: `reports/player_<First>_<Last>_<LocationSlug>_<YYYYMMDD>.docx` (always date-stamped — ratings and match history change over time).

Share via a `computer://` link once written.

## Failure modes

| Signal | Required behavior |
|--------|-------------------|
| Search returns 0 rows | Tell the user, suggest checking spelling or providing a location. Do not write a report. |
| Search returns >1 row and `--location` not set | Ask the user via `AskUserQuestion`, one option per candidate. |
| Search returns >1 row and `--location` matches multiple | Ask the user; do not auto-pick. |
| Profile page returns 404 | Stop. The `&s=N` on the search page was likely stale — ask the user to retry. |
| Rate-limit / CAPTCHA / 429 | Stop. Do not retry aggressively. Suggest trying again later. |
| Year page has no matches | Skip that year; continue with remaining years. |
| Rating cell shows `-----` | Coerce to `None`. Do not treat as `0`. |
| Zero matches in lookback window | Produce a report with profile summary only and note "No matches in the last {MONTHS} months". |
| Rating jumps > 0.3 between consecutive matches | Flag in TL;DR as a possible self-rated bump, regardless of `C`/`S`. |

## Validation

Before handing the `.docx` to the user:
- Confirm the match log row count equals the filtered match count.
- Confirm every W/L cell is shaded (green/red).
- Confirm the rating trend slope label matches the numeric slope sign.
- Confirm no numeric field shows `-----`, `None`, or a raw HTML fragment.
- Confirm the filename is date-stamped and lives under `reports/`.

If the generator raises an error, treat it as a failed build and fix the parsing issue before delivering.
