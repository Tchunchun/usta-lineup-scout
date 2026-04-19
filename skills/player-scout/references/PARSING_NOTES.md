# Parsing Notes — player-scout

Tennisrecord.com HTML quirks specific to the per-player search, profile, and match-history pages. Load only when debugging a parsing failure or handling one of these edge cases.

## Search endpoint

- POST with `firstname` and `lastname` form fields to `/adult/search.aspx`. GET will not return results.
- The result table has columns: `Player Name | Location | Gender | NTRP | Updated`.
- Each row's `<a>` points to `/adult/profile.aspx?playername=<Name>[&s=N]`. **Use the exact `href` from the search row**; do not rebuild it from the name — the `&s=N` disambiguator is load-bearing for common names.

## Location aliases

A small alias table handles common USTA section shortcuts. Substring-match against the Location column after expansion:

| Input alias | Matches |
|-------------|---------|
| `PNW` / `Pacific NW` | `WA`, `OR`, `ID` |
| `NorCal` | Northern California cities |
| `SoCal` | Southern California cities |
| `PacNW` | Same as `PNW` |

When the alias expands to multiple candidates, ask the user to pick — do not auto-select.

## Profile page

- Current rating string format: `<level> <type>` e.g. `3.5 C`, `3.5 S`. Any suffix other than `C` or `S` → treat as unknown.
- Estimated Dynamic Rating is to 4 decimals with an "as of" date in `MM/DD/YYYY` format — convert to ISO.
- Projected Year End Rating may render as `—` (em dash); coerce to `None`.

## Match-history page structure

Each match is rendered as a small 4-row table:

| Row | Fields |
|-----|--------|
| 1 | `Match Date | Court (S1/S2/D1/D2/D3) | League + Level` |
| 2 | `My Team | W or L | Opponent Team` |
| 3 | `Partner (blank for singles) | Score | Opponent(s) with (dynamic rating)` |
| 4 | `Match difficulty number | (blank) | Post-match dynamic rating` |

For doubles, row 3's right cell contains two opponents separated by `<br>`, each with their rating in parentheses.

### Field conventions

- Dates are `MM/DD/YYYY` → convert to ISO (`YYYY-MM-DD`).
- `Match:` value in row 4 is the **match difficulty** — effective opponent rating faced. Use this as the opponent-strength metric.
- `Rating:` value in row 4 is the **post-match dynamic rating**. A red color (`color:#DD0000`) suggests the rating is falling; green (`color:#00DD00`) suggests it held or rose. Record the color as a hint but rely on the numeric series for the trend label.
- Missing ratings render as `-----`. Coerce to `None`, never to `0`.
- Scores are space-separated sets in the final string; inside the HTML cell they may be split by `<br>`.

## Deduplication

Matches near the year boundary can appear on two year pages. Dedupe on `(date, court, sorted opponent names)`.

## Performance and politeness

- Parallelize year pulls — one WebFetch per year, issued concurrently.
- Tennisrecord.com is a small volunteer-run site. If scouting several players back-to-back, add a small delay between runs. Stop immediately on 403, 429, or CAPTCHA — do not retry aggressively.
