# USTA Scout

Small workspace for building USTA opponent scouting reports from public `tennisrecord.com` data.

This repo includes two skills:

- **usta-team-scout** — whole-team `.docx` scouting report
- **usta-player-scout** — single-player `.docx` scouting report

## What It Does

Both skills pull from TennisRecord and generate `.docx` reports containing:

- roster table with dynamic ratings and rating types (`C` = computer, `S` = self-reported)
- completed match lineups with court-by-court results
- players who have not yet appeared in a completed match
- strategy notes by court
- optional manual recent-match ingestion from JSON when TennisRecord is behind

## Repo Structure

```text
skills/
  usta-team-scout/
    SKILL.md                    Skill instructions
    scripts/generate_report.py  Team report generator
    references/                 Parsing notes and report style guide
    evals/evals.json            Sample eval prompts
  usta-player-scout/
    SKILL.md                    Skill instructions
    scripts/player_report.py    Offline player report renderer
    references/                 Parsing notes and report style guide
    evals/evals.json            Sample eval prompts
```

## Requirements

- Python 3.10+
- Network access to `tennisrecord.com` for the team skill and for player-scout data collection via `fetch_webpage`

Install Python packages:

```bash
python3 -m pip install --user python-docx beautifulsoup4 lxml
```

## Usage

**Team report:**

```bash
python3 skills/usta-team-scout/scripts/generate_report.py --team "TEAM NAME" --year 2026
```

**Team report with manual recent matches:**

```bash
python3 skills/usta-team-scout/scripts/generate_report.py --team "TEAM NAME" --year 2026 --manual-matches path/to/manual_matches.json
```

**Player report:**

```bash
python3 skills/usta-player-scout/scripts/player_report.py --input path/to/player_data.json
```

Reports are written to `reports/` at the repo root (gitignored).

## Notes

- Profile links include `&s=` disambiguators where needed to handle players sharing a name.
- Player profile URLs must use `%20` for spaces, not `+`.
- Only completed matches are included in reports.
- Unsupported rating suffixes are normalized to unknown `(—)`.
- Intended for private team prep and small-scale manual use.
