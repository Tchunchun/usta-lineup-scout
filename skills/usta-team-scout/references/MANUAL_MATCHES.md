# Manual Match JSON Schema

Use this file when TennisRecord has not yet posted a recent match and you need to include the result in the generated team report.

Pass the JSON file to the generator with:

```bash
python3 scripts/generate_report.py \
  --team "<TEAM NAME>" \
  --year <YEAR> \
  --manual-matches path/to/manual_matches.json
```

## Required shape

Top-level value: JSON array of match objects.

Each match object must contain:

```json
{
  "date": "4/25/2026",
  "site": "Harbor Square Athletic Club",
  "opponent": "ETC-Fresh off the Court-Zhu/Walsh",
  "courts": [
    {
      "court": "S1",
      "team_players": ["Player A"],
      "opponent_players": ["Opponent A"],
      "score": "6-2 6-4",
      "result": "W"
    },
    {
      "court": "D1",
      "team_players": ["Player B", "Player C"],
      "opponent_players": ["Opponent B", "Opponent C"],
      "score": "6-4 4-6 1-0",
      "result": "L"
    }
  ]
}
```

## Field rules

- `date`: display string written into the report.
- `site`: display string written into the report.
- `opponent`: opponent team name as it should appear in the report.
- `courts`: array of court objects.
- `court`: one of `S1`, `S2`, `D1`, `D2`, `D3`.
- `team_players`: one name for singles, two names for doubles.
- `opponent_players`: one name for singles, two names for doubles.
- `score`: winner-first score string, matching TennisRecord display convention.
- `result`: `W` or `L` from the scouted team's perspective.

## Behavioral notes

- The generator fetches opponent DR and rating type automatically when possible.
- Manual matches are appended to scraped completed matches.
- The scouted team's season record and local singles/doubles splits are updated before writing the roster table.
- Manual matches participate in wildcard removal and strategy calculations.

## Failure cases

- Missing required keys: the generator should fail fast.
- Invalid court labels: the generator should fail fast.
- Names absent from the scouted-team roster: the generator keeps the match row, but only roster members get record updates.