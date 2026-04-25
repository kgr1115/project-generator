<!--
This is the canonical structure for a generated CLAUDE.md.

The skill at .claude/skills/generate-claudemd/SKILL.md fills the {{PLACEHOLDERS}} from
interview answers and STRIPS any section whose content is empty / "N/A" / trivially generic.

Sections marked [optional] are dropped entirely if the user skipped them. Sections marked
[required] always appear (or the generator stops to re-ask).

Target output: under 200 lines total. Trim ruthlessly.

Comment blocks like this one are stripped before writing — block-level HTML comments are
removed by Claude Code's loader anyway, but we strip them here too for human readability.
-->

# {{PROJECT_NAME}} — standing brief

> Read this first when working in this repository.

## What this project is

[required — 1–3 sentences]

{{ONE_LINE_DESCRIPTION}}

{{WHO_ITS_FOR_SENTENCE}}

## What this project is NOT

[required — bulleted list of 2–5 scope walls. The most load-bearing section in the file.]

{{SCOPE_NOTS_BULLETS}}

## How to run it

[required for retrofit; optional for greenfield if not yet decided]

```
{{RUN_COMMANDS}}
```

[Include: install, dev/run, test. One line each. No prose.]

## Tech stack

[required — terse]

- **Language:** {{LANGUAGE}}
- **Runtime:** {{RUNTIME}}
- **Framework:** {{FRAMEWORK}}
- **Package manager:** {{PACKAGE_MANAGER}}
- **Test runner:** {{TEST_RUNNER}}

[Strip lines that are N/A. Don't list 10 dependencies — that's what `@package.json` is for.]

## Conventions

[optional — only deviations from language defaults]

{{CONVENTION_BULLETS}}

[Examples of good entries:
- "Use 4-space indent for JS — historical, don't change."
- "Imports ordered: stdlib → third-party → local. Enforced by ruff."
- "All functions exported from `index.ts`; no deep imports."

Skip the section entirely if no real deviations exist.]

## Workflow

[optional — skipped for solo projects]

- **Branches:** {{BRANCH_STRATEGY}}
- **Commits:** {{COMMIT_STYLE}}
- **PRs:** {{PR_CONVENTION}}

## AI collaboration rules

[required — this is what most CLAUDE.md files miss]

{{AI_RULES_BULLETS}}

[Examples of good entries:
- "**IMPORTANT:** Never push without my explicit go-ahead."
- "**IMPORTANT:** Never `git add -A` — stage files explicitly by name."
- "Never skip commit hooks (`--no-verify`, `--no-gpg-sign`)."
- "For changes touching `src/billing/`, use plan mode first."
- "Don't add new dependencies without confirming with me."

Each rule needs to be specific enough to verify. "Be careful" is not a rule.]

## Gotchas / known landmines

[required if user supplied any; the highest-value section by far]

{{GOTCHA_BULLETS}}

[Examples of good entries:
- "Timestamps in `events.log` are UTC; everywhere else uses local time."
- "`os.replace()` only — `shutil.move()` fails silently on cross-volume on Windows."
- "After `docx → pdf` conversion, verify `pdf.stat().st_size > 0` before deleting source."
- "The `users` collection has a unique index on email; INSERT will throw, not upsert."

Each entry is a fact + the wrong assumption it prevents. Add new ones as they're discovered.]

## Glossary

[optional — domain terms only, skip if not applicable]

{{GLOSSARY_BULLETS}}

## Data sensitivity

[optional — included only if non-trivial]

{{DATA_SENSITIVITY_PARAGRAPH}}

[Example: "This repo holds real customer emails in `data/originals/`. Never push the originals; the personal-data guard at `blocked-paths.txt` enforces this."]

## References

[optional — external docs, dashboards, trackers]

{{REFERENCE_BULLETS}}

[Use `@path` imports for in-repo files. Use plain URLs (or just names + locations) for external resources — Claude can't follow URLs unauthenticated, but the human reading this can.]

## See also

[optional — only if multiple in-repo docs exist]

- `@README.md` — public-facing overview
{{ADDITIONAL_SEE_ALSO}}
