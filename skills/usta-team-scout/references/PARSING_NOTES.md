# Parsing Notes — usta-team-scout

Tennisrecord.com HTML quirks and edge cases accumulated from prior runs. Load only when debugging a parsing failure or when the skill is about to hit one of these cases.

## URL and disambiguation

- **Profile `&s=N` is load-bearing.** Some players share names. Always use the exact `href` that appears on the team or match page; do not rebuild the URL from the raw player name.
- **Team `&s=N`** — same idea at the team level. If the team profile page shows the wrong league, increment `&s` until the correct league appears.
- **tennisrecord.com match IDs ≠ TennisLink match IDs.** If the user gives a TennisLink ID, ignore it. Use the `mid=` parameter from the team profile page.

## Roster vs match-page name matching

When cross-referencing roster names against match participants for wildcard detection:
1. Match on `href` first — most reliable, handles `&s=` disambiguators cleanly.
2. Fall back to normalized-name: strip diacritics, lowercase, collapse whitespace, drop middle initials.
3. If still no match, treat as wildcard and note the ambiguity in the report footer.

## Performance

- Fetch profiles in parallel. For a typical 15–20 player roster plus ~15 opponent match participants, sequential fetches take several minutes; one batched parallel WebFetch call completes in seconds.
- Cache within a single run. If the same `href` is referenced from multiple matches, fetch once.

## Data hygiene

- **Unknown rating suffixes.** Only `C` and `S` are first-class. Any other suffix letter (e.g. `A`, `T`, `M`) → normalize to unknown `(—)`.
- **Missing DR.** Render as `—`, not `0.0` or `None`.
- **Forfeit matches.** The court row is present but lacks a score. Mark `FORFEIT` in the result cell; do not invent opponent names.

## Score display convention

TennisRecord (and match screenshots from TennisLink) display scores **match-winner-first**: the player or team that won the court always has their game count listed first in every set, even for sets they lost.

Examples:
- A court winner who won 6-3, 6-2 → displayed as `6-3 6-2`
- A court winner who won 7-5, 4-6, 6-3 → displayed as `7-5 4-6 6-3` (they lost the second set, but still show their score first in every set)
- Supertiebreak: always displayed as `1-0` for the winner, never the actual score (e.g. `10-7`)

**When parsing a screenshot:** identify which player/team won each court first, then read scores left-to-right as that winner's game count. If the scouted team was the visiting team, their score column may appear on the right in the screenshot — invert the column order mentally before recording.

**In the match table:** always record the scouted team's score first (left column), consistent with how TennisRecord lays out its match result pages.

## Rate limiting

Tennisrecord.com is a small volunteer-run site. Don't hammer it. Keep parallel fetch batches reasonable (one team + one run's worth of profiles at a time). If the site returns a 403, 429, or CAPTCHA, stop — do not retry aggressively.
