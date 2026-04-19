# Report Style Reference — usta-team-scout

Visual conventions for the team scouting `.docx` report. Load only when generating the report.

## Symbol legend

| Symbol | Meaning |
|--------|---------|
| ◆ | DR ≥ 3.0 player (strong) |
| ★ | Not yet played (wildcard) |
| ⚠ | Self-rated (S) — treat as unknown |
| (C) | Computer Rated |
| (S) | Self Rated |
| (—) | Unknown / no rating available |

## Row and cell conventions

| Element | Convention |
|---------|-----------|
| DR ≥ 3.0 player | Blue shading + **bold** + ◆ symbol |
| Self-rated player | `(S)` after name, red cell in Rating column, ⚠ symbol |
| Not yet played | Yellow row shading + `NOT YET PLAYED ★` in Status column |
| Scouted team wins a court | Green result cell |
| Scouted team loses a court | Red result cell |
| Forfeit | `FORFEIT` in result cell, neutral shading |
| C/S in match tables | `Player Name (C)` or `Player Name (S)` — always full name |

## Color palette (hex)

| Name | Hex | Usage |
|------|-----|-------|
| Navy | `1F3864` | Section headers, header cell fill |
| Light blue | `D6E4F0` | DR ≥ 3.0 cell shading |
| Mid blue | `2E75B6` | Accent lines, legend borders |
| Win green | `E2EFDA` | Result cell for scouted-team wins |
| Loss red | `FCE4D6` | Result cell for scouted-team losses |
| Warning yellow | `FFF2CC` | Wildcard row shading |
| White | `FFFFFF` | Header text on navy fill |

## Column layouts

**Full roster table:**
`# | Player | DR | Rating (C/S) | Season Record | Singles | Doubles | {YEAR} Status`

**Per-match table:**
`Court | Scouted Team Player | DR | Result | Opponent Player | DR`

**Doubles rows:** format as `Player A (C) / Player B (S)`. Each partner gets their own rating-type suffix. Never abbreviate names.

## Style goals

A captain should be able to skim the full report and pick lineups in under 5 minutes. Avoid visual noise — use shading to signal status, not to decorate.
