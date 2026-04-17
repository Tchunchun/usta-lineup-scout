# USTA Scout

Small workspace for building USTA opponent scouting reports from public `tennisrecord.com` data.

This repo currently includes:

- a local Codex skill at `skills/usta-scout/`
- a Python report generator that produces `.docx` scouting reports
- sample scouting outputs and reference materials for the 2026 season

## What It Does

The scouting workflow pulls:

- team roster data
- dynamic ratings (DR)
- rating types (`C` or `S`, with anything else treated as unknown)
- completed match lineups
- court-by-court results
- players on the roster who have not appeared in a completed match yet

It then generates a `.docx` scouting report with:

- title block and league metadata
- roster table
- one table per completed match
- strategy notes by court

## Repo Structure

```text
skills/usta-scout/
  SKILL.md              Codex skill instructions
  evals/evals.json      Sample eval prompts
  generate_report.py    Local report generator
```

## Requirements

- `python3`
- network access to `tennisrecord.com`

Python packages:

- `python-docx`
- `beautifulsoup4`
- `lxml`

Install them with:

```bash
python3 -m pip install --user python-docx beautifulsoup4 lxml
```

## Usage

Generate a report with:

```bash
python3 skills/usta-scout/generate_report.py --team "TEAM NAME" --year 2026
```

Example:

```bash
python3 skills/usta-scout/generate_report.py --team "TEAM NAME HERE" --year 2026
```

Optional custom output path:

```bash
python3 skills/usta-scout/generate_report.py \
  --team "TEAM NAME HERE" \
  --year 2026 \
  --output "./Scouting_Report.docx"
```

## Notes

- The script uses the exact TennisRecord profile links found on team and match pages so it can handle players that require `&s=` disambiguators.
- Only completed matches are included in the report.
- Unsupported rating suffixes are normalized to unknown `(—)` rather than treated as a special rating class.
- The current workflow is best suited for private team prep and small-scale manual use.

## Future Improvements

- add caching so repeated runs are faster
- parallelize profile fetches
- make strategy notes more opponent-specific
- add automated regression checks for parsing changes on TennisRecord pages
