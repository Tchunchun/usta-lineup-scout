# Agent Instructions

## Project

USTA opponent scouting reports from `tennisrecord.com`. Two skills under `skills/`:

- **usta-team-scout** — whole-team `.docx` report
- **usta-player-scout** — single-player `.docx` report

## Dependencies

- Python 3.10+
- `python-docx`, `beautifulsoup4`, `lxml`

```bash
python3 -m pip install --user python-docx beautifulsoup4 lxml
```

## File-Scoped Commands

| Task | Command |
|------|---------|
| Team report | `python3 skills/usta-team-scout/scripts/generate_report.py --team "NAME" --year 2026` |
| Player report | `python3 skills/usta-player-scout/scripts/player_report.py --input path/to/player_data.json` |

## Key Conventions

- Reports are always written to `reports/` at repo root, never inside `skills/`
- Skill directories follow the Agent Skills Specification (`SKILL.md` + `scripts/`, `references/`, `evals/`)
- See `SKILL_GUIDELINES.md` for authoring rules
- Rating types: `C` (computer), `S` (self-reported), anything else → unknown
- All HTML parsing targets `tennisrecord.com`; see `references/PARSING_NOTES.md` in each skill

## Commit Attribution

AI commits MUST include (use whichever agent produced the code):

```
Co-Authored-By: Claude <noreply@anthropic.com>
Co-Authored-By: GitHub Copilot <noreply@github.com>
```
