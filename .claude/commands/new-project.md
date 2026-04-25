---
description: Bootstrap a new project with a tailored CLAUDE.md, optional agent improvement pipeline, and either a single-repo or paired private/public layout. Replaces ai-pipeline-scaffold's /begin. Triggered by /new-project, /new, or natural-language phrases like "start a new project", "let's begin", "bootstrap a new repo".
argument-hint: [optional project name — e.g. "acme-crm" — omit to be prompted]
---

The user wants to bootstrap a new project. Invoke the `new-project` skill with the argument `$ARGUMENTS`.

The skill will:
1. Resolve the project name (use `$ARGUMENTS` or ask).
2. Ask the layout question (`single` vs `dual` repo) — no default.
3. Ask whether to include the agent improvement pipeline (default `yes`).
4. Run the full project-intent interview via the `interview-project-intent` sub-skill.
5. Preview the plan: target paths, what gets copied, scope rules pulled from the interview, what gets pruned if the pipeline was opted out.
6. Confirm with the user, then execute:
   - Create `<name>-private/` (and `<name>-public/` for dual layout) as siblings of the parent directory.
   - Copy `templates/private-scaffold/` into the private side, with placeholders substituted.
   - Copy `templates/public-scaffold/` into the public side (dual only).
   - Replace placeholder scope rules in `architect.md` and `architect-review/SKILL.md` with the user's interview answers.
   - Seed `blocked-paths.txt` from data sensitivity answers.
   - Write a tailored `CLAUDE.md` in each created repo (private and public versions differ — public-side omits private-only rules).
   - `git init` in each created repo. No remote is added.
7. Report paths, next steps, and what was pruned.

Follow the skill exactly — do not improvise the file layout, skip the preview phase, or substitute placeholders that haven't been answered.
