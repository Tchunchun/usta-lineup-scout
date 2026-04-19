# USTA Scout

Small workspace for building USTA opponent scouting reports from public `tennisrecord.com` data.

This repo includes two skills:

- **usta-team-scout** — whole-team `.docx` scouting report
- **player-scout** — single-player `.docx` scouting report

## What It Does

Both skills pull from TennisRecord and generate `.docx` reports containing:

- roster table with dynamic ratings and rating types (`C` = computer, `S` = self-reported)
- completed match lineups with court-by-court results
- players who have not yet appeared in a completed match
- strategy notes by court

## Repo Structure

```text
skills/
  usta-team-scout/
    SKILL.md                    Skill instructions
    scripts/generate_report.py  Team report generator
    references/                 Parsing notes and report style guide
    evals/evals.json            Sample eval prompts
  player-scout/
    SKILL.md                    Skill instructions
    scripts/player_report.py    Player report generator
    references/                 Parsing notes and report style guide
    evals/evals.json            Sample eval prompts
```

## Requirements

- Python 3.10+
- Network access to `tennisrecord.com`

Install Python packages:

```bash
python3 -m pip install --user python-docx beautifulsoup4 lxml
```

## Usage

**Team report:**

```bash
python3 skills/usta-team-scout/scripts/generate_report.py --team "TEAM NAME" --year 2026
```

**Player report:**

```bash
python3 skills/player-scout/scripts/player_report.py --player "First Last" --area "AREA" --year 2026
```

Reports are written to `reports/` at the repo root (gitignored).

## Notes

- Profile links include `&s=` disambiguators where needed to handle players sharing a name.
- Only completed matches are included in reports.
- Unsupported rating suffixes are normalized to unknown `(—)`.
- Intended for private team prep and small-scale manual use.
