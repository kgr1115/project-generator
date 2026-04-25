# {{PROJECT_NAME}}

> Standing brief for Claude Code working in this repository.

## What this project is

{{PROJECT_DESCRIPTION}}

## Who uses it

{{END_USER_DESCRIPTION}}

## Scope (what this project IS and IS NOT)

**{{PROJECT_NAME}} IS:**
- <fill in — e.g. "a CLI that does X", "a web app that does Y">
- <runtime environment — e.g. "runs on user's laptop via `python main.py`">
- <cost model — e.g. "free + local only; no paid services">

**{{PROJECT_NAME}} IS NOT:**
- <fill in — e.g. "not a SaaS, no hosted services, no accounts/auth">
- <fill in — e.g. "not multi-tenant, not a platform">

<!--
EXAMPLE — a filled-in scope block for a hypothetical "acme-crm" project.
Delete this block once your own scope above is filled in.

**acme-crm IS:**
- A local CRM for a solo founder tracking deals and contacts.
- Runs on the user's laptop via `python -m acme_crm` — no server.
- Free + local only; only budgeted cost is LLM API tokens.
- Single-user, single-tenant — contact list is personal, lives on disk.

**acme-crm IS NOT:**
- A SaaS product. No hosted deployment, no accounts, no telemetry.
- A multi-user platform — no auth, no sharing, no tenant isolation.
- A general-purpose CRM — tuned to this user's specific workflow.
-->


## How to work here

- **Before editing code**: read the relevant agent profile in `.claude/agents/` and the matching skill in `.claude/skills/` — the project's playbooks live there.
- **When adding a new workflow**: prefer a new skill (`.claude/skills/<name>/SKILL.md`) over adding another agent. Skills are the playbook library; agents are the execution contexts that run them.
- **When running the project's core workflow**: use the `/run` slash command (or whatever command(s) this project defines — see `.claude/commands/`).

## Absolute constraints

1. **Never** take irreversible real-world actions (submissions, emails, posts, deploys, data mutations) without explicit user approval on that specific item.
2. **Never** run `--dangerously-skip-permissions`. Use scoped `permissions.allow` in `.claude/settings.json` instead.
3. **Never** `git add -A` or `git add .`. Stage files explicitly by name.
4. **Never** push without the user's go-ahead.
5. **Never** mutate user-owned source material unilaterally — treat originals as read-only unless the user explicitly authorizes edits.

## Project-specific notes

<!-- Add anything Claude should know before touching this repo — data layouts, known gotchas, file naming conventions, external service endpoints, etc. -->
