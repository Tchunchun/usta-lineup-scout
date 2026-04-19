# Skill Authoring Guidelines

Guidance for authoring and maintaining skills in this repository, and by extension any skill following the Agent Skills format.

**Canonical reference:** [Agent Skills Specification — agentskills.io/specification](https://agentskills.io/specification). This document defers to that spec on all format-level questions and layers repository-specific conventions on top. Where the spec and this document conflict, the spec wins.

The tone here matches the spec's: plain prose, "must" for hard requirements, "should" for strong recommendations, "may" for options. No shouting.

---

## 1. Scope

These guidelines apply to every skill under `skills/` in this repository (currently `skills/usta-team-scout/`) and to any new skill added later. Project-agnostic rules are marked *(general)*; repo-specific rules are marked *(usta-lineup-scout)*.

---

## 2. Directory layout

### 2.1 General

A skill is a directory containing, at minimum, a `SKILL.md` file. The directory name must exactly match the `name` field in the frontmatter.

Recommended layout, matching the spec's optional directories:

```
skills/<skill-name>/
├── SKILL.md          # required: metadata + instructions
├── scripts/          # executable code
├── references/       # on-demand documentation
├── assets/           # templates, static data
└── evals/evals.json  # evaluation cases
```

Executable code should live in `scripts/`. Documentation loaded only when a task needs it should live in `references/`. Templates and static data should live in `assets/`. File references inside `SKILL.md` should be relative to the skill root and kept one level deep.

### 2.2 usta-lineup-scout

All skills must live under `skills/<skill-name>/` at the repo root. The report generator should live at `skills/<skill-name>/scripts/generate_report.py`, not at the skill root. Generated outputs (`.docx` files, caches) must be written to `reports/` at the repo root, never inside the skill directory. `.DS_Store` and editor state must be gitignored.

---

## 3. `SKILL.md` frontmatter

Authoritative field definitions live in the [spec](https://agentskills.io/specification). What follows is additive guidance.

### 3.1 `name`

Must be 1–64 characters, lowercase `a–z`, digits, and hyphens only. Must not start or end with a hyphen and must not contain consecutive hyphens. Must match the parent directory name.

### 3.2 `description`

Must be 1–1024 characters. Per the spec, it should describe both what the skill does and when to use it, and should include specific keywords that help agents identify relevant tasks.

Beyond the spec, this repo recommends:

- Lead with the action verb and primary output type (e.g., "Generate a .docx opponent report…"). Burying the output at the end of a long parenthetical hurts activation accuracy.
- Include three to six high-signal trigger phrases a user would actually say.
- Include a short out-of-scope clause ("Do not use for…") when false-positive activation is plausible.
- Write as a single-line YAML string unless a folded scalar is genuinely needed; folded scalars collapse whitespace inconsistently across parsers.
- Avoid repeating the same domain token more than twice — it wastes the character budget.

### 3.3 Optional fields

The spec defines `license`, `compatibility`, `metadata`, and (experimental) `allowed-tools`. Prefer setting them explicitly rather than relying on defaults:

- `license` — should be set. Use "Proprietary" for private or internal skills.
- `compatibility` — should be set when the skill requires a specific runtime, package, or network access. Max 500 characters.
- `metadata.version` — should be set and bumped on any breaking change to skill behavior or output format.
- `metadata.author` — optional.
- `allowed-tools` — optional and experimental. If set, it must accurately reflect the tools the skill actually uses.

### 3.4 usta-lineup-scout frontmatter minimum

Every skill in this repo should include at least:

```yaml
---
name: <skill-name>
description: <single-line, action-first, with triggers and out-of-scope>
license: Proprietary
compatibility: <runtime + network requirements>
metadata:
  version: "<semver>"
  author: usta-lineup-scout
---
```

---

## 4. Body content

### 4.1 Length and progressive disclosure

Per the spec, the agent loads the full `SKILL.md` body when the skill is activated, so every token counts. Keep the body under 500 lines and ideally under 5,000 tokens.

Material only needed *while performing* the task — style tables, hex palettes, edge-case catalogs, tips from prior runs — should live in `references/` and be linked, not inlined. Reference files should be small and focused (one topic per file) so the agent can load just what it needs.

### 4.2 Required sections

Every `SKILL.md` in this repo should include, in this order:

1. **Overview** — one paragraph: what the skill does and what it produces.
2. **When to use / When not to use** — explicit trigger phrases and explicit out-of-scope cases.
3. **Inputs** — every parameter the skill expects, with defaults.
4. **Steps** — numbered, imperative, one action per step.
5. **Outputs** — where files land, naming convention, and what success looks like.
6. **Failure modes** — how to recognize and handle each failure case.
7. **Validation** — how the agent verifies the output before handing it to the user.

### 4.3 Writing style

Steps should be imperative ("Fetch the team page", not "The team page should be fetched") and should name the tool the agent is expected to use when it matters ("Fetch all profiles in a single parallel WebFetch call").

Time- or season-specific strings (years, versions) should be parameterized (`{YEAR}`, `{SEASON}`) rather than hardcoded, unless the hardcoded value is the point of the skill.

Paths in examples must be relative to the skill root, and the doc must not assume a particular invocation working directory.

### 4.4 Failure modes

Every skill that calls external services should document, at minimum:

- Network / 4xx / 5xx response handling.
- Rate-limit or CAPTCHA response handling.
- Malformed or schema-changed upstream data.
- Empty-result cases (no matches, no roster, etc.).
- Ambiguous input (e.g., multiple teams matching the same name).

For each, state the detection signal and the required agent behavior (retry, ask user, abort, degrade gracefully).

---

## 5. Scripts

### 5.1 General

Scripts should live in `scripts/` and be invoked via relative paths from the skill root. They should print a clear error message and exit non-zero on any recoverable failure, and they should validate their inputs before doing network or filesystem work. Scripts should not hardcode output paths; accept an `--output` or similar flag. Dependencies should be documented in a header comment or a nearby `requirements.txt` if anything beyond the standard library is needed.

### 5.2 usta-lineup-scout

Python scripts should target Python 3.10+. Network scraping scripts should be polite — sequential or small-batch parallel, with an identifying User-Agent and a timeout on every request. Scripts should not write outside the repo's `reports/` folder unless the caller explicitly passes an absolute path. Scripts that generate documents should do structural validation (e.g., "roster has ≥1 row", "every court row has both teams") before writing the file.

---

## 6. References and assets

Files in `references/` should be loadable on demand and should not be required to understand the skill. Files in `assets/` should be static and non-executable: templates, hex palettes as data, sample inputs. Binary assets should be kept small; link to an external source for anything over roughly 1 MB.

---

## 7. Evals

### 7.1 General

Every skill should have an `evals/evals.json` with at least six entries covering:

- Happy path (at least two).
- Edge cases specific to the skill's domain.
- At least one explicit negative case the skill should *not* trigger on.
- At least one failure-mode case (e.g., 404, empty result).

Each eval entry should include a pass criterion, not just a prose description. A pass criterion is a set of checkable assertions (file exists, contains expected strings, schema matches).

### 7.2 Recommended eval schema

```json
{
  "skill_name": "<skill-name>",
  "evals": [
    {
      "id": 1,
      "prompt": "<exact user prompt>",
      "inputs": { "...": "..." },
      "expected_output": "<prose summary>",
      "pass_criteria": [
        "Output file exists at reports/...",
        "Contains roster table with N rows",
        "Strategy section mentions court S1, S2, D1, D2, D3"
      ],
      "tags": ["happy-path"]
    }
  ]
}
```

Valid `tags` values: `happy-path`, `edge-case`, `negative`, `failure-mode`.

---

## 8. Versioning and change management

Breaking changes to output format, CLI flags, or expected prompts should bump `metadata.version` (major). Additive changes should bump the minor version. Any change that alters skill activation behavior (the `description` field) should be validated by running the eval suite before merging. Each PR that modifies a skill should include a line in the commit message stating which evals were re-run.

---

## 9. Validation

Before merging any skill change, the author should:

1. Run `skills-ref validate ./skills/<skill-name>` (see the spec's [Validation section](https://agentskills.io/specification)) and fix any errors.
2. Confirm `SKILL.md` is under 500 lines.
3. Confirm the `description` is under 1024 characters.
4. Re-run the eval suite and record the result.
5. Verify the skill directory name matches the `name` field.

---

## 10. Review checklist

Use this when reviewing a new or modified skill.

```
Frontmatter
[ ] name matches parent directory and spec rules
[ ] description is single-line, action-first, ≤1024 chars
[ ] description contains triggers AND out-of-scope note
[ ] license set
[ ] compatibility set if any non-trivial runtime requirement
[ ] metadata.version set (semver)

Body
[ ] Overview, When/When-not, Inputs, Steps, Outputs, Failure modes, Validation — all present
[ ] ≤500 lines, ≤~5000 tokens
[ ] Steps are imperative and tool-aware where relevant
[ ] Year / season / version strings are parameterized
[ ] Paths are relative to skill root
[ ] Style tables / hex palettes / tips moved to references/

Layout
[ ] scripts/ used for executables
[ ] references/ used for on-demand docs
[ ] evals/evals.json present with ≥6 entries covering all tag categories
[ ] No .DS_Store / editor state committed

Behavior
[ ] skills-ref validate passes
[ ] Eval suite re-run and results recorded in PR
[ ] Script exits non-zero with a clear message on every failure mode listed in SKILL.md
```

---

## 11. Applying these guidelines to `usta-team-scout`

A non-exhaustive list of changes the current `skills/usta-team-scout` needs to conform:

- Move `generate_report.py` into `scripts/`.
- Move the "Visual conventions", "Color values", "Doubles rows", and "Tips from prior runs" sections into `references/REPORT_STYLE.md` and `references/PARSING_NOTES.md`.
- Rewrite the `description` as a single-line string, lead with the action, drop the long parenthetical example team name, and add an out-of-scope clause.
- Add `license`, `compatibility`, and `metadata.version` to the frontmatter.
- Parameterize `{YEAR}` everywhere "2026" appears as a column header or default.
- Add a "Failure modes" section covering 404 team URL, CAPTCHA / rate-limit, profile missing DR, forfeit matches, and multi-league `&s=N` ambiguity.
- Expand `evals/evals.json` to at least six entries with `pass_criteria` and `tags`, including at least one negative case and one failure-mode case.
- Document the wildcard name-matching strategy (href first, then normalized-name fallback).

---

## 12. References

- [Agent Skills Specification — agentskills.io/specification](https://agentskills.io/specification) — canonical format, field definitions, validator.
- [skills-ref validator (GitHub)](https://github.com/agentskills/agentskills/tree/main/skills-ref) — referenced in §9.
