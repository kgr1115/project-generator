# project-generator

A unified bootstrap-and-brief tool for Claude Code projects. Interviews you about a project's intent, scope, conventions, and gotchas, then either scaffolds a new project or generates a tailored `CLAUDE.md` for an existing one.

This is the successor to `ai-pipeline-scaffold`'s `/begin` flow — same paired-fork option, same agent improvement pipeline, but with a deeper interview that produces real CLAUDE.md content (not placeholders).

## Two entry points

| Command | Use when | What you get |
|---|---|---|
| **`/new-project [name]`** | Starting a new project from nothing | A scaffolded repo (single or paired private/public), optionally seeded with the agent improvement pipeline, plus a tailored `CLAUDE.md` based on your interview answers. `git init`'d, no remote. |
| **`/generate-claudemd [target]`** | Existing repo missing a useful `CLAUDE.md` | A tailored `CLAUDE.md` written into the target directory. Pre-scans manifests to skip questions it can answer itself. No directory scaffolding. |

Both share one interview skill (`interview-project-intent`) so question wording stays consistent.

## Why this exists

`ai-pipeline-scaffold`'s `/begin` does the directory work but leaves you with placeholder scope rules and a placeholder `CLAUDE.md`. You still have to fill in the architect's scope answers, the personal-data guard, and write CLAUDE.md content yourself. Three problems with that:

1. **The deepest decisions get made when you're tired.** After scaffolding 12 files, you're least equipped to articulate "what should this project never become."
2. **The same answers go in multiple places.** Scope walls land in `architect.md` AND `CLAUDE.md` AND `architect-review/SKILL.md`. Asking once and propagating is the whole point.
3. **Most projects never get a real CLAUDE.md.** They keep the placeholder template, which gives Claude almost nothing to work with.

`project-generator` flips it: deep interview first, then everything else flows from those answers. One question's answer can feed up to four files automatically.

## Quick start

```
cd C:\Projects\project-generator
# Open Claude Code, then:
/new-project my-new-project
# OR, in an existing repo:
/generate-claudemd
```

For `/new-project`, you'll be asked:
1. Layout — `dual` (paired private + public) or `single`
2. Pipeline — include the agent improvement pipeline?
3. Then 12–15 interview questions (project intent, stack, conventions, AI rules, gotchas)
4. Plan preview + confirm
5. Disk writes + `git init`

For `/generate-claudemd`, you'll be asked:
1. About 6–10 interview questions (manifests pre-fill what they can)
2. Preview the assembled `CLAUDE.md`
3. Iterate or approve
4. Write to `<target>/CLAUDE.md`

## What's in this repo

```
project-generator/
├── README.md                                    This file
├── CLAUDE.md                                    Standing brief for working in THIS repo
├── .claude/
│   ├── commands/
│   │   ├── new-project.md                       /new-project entry point
│   │   └── generate-claudemd.md                 /generate-claudemd entry point
│   └── skills/
│       ├── new-project/SKILL.md                 Full bootstrap playbook
│       ├── generate-claudemd/SKILL.md           Retrofit-only playbook
│       └── interview-project-intent/SKILL.md    Shared interview engine (called by both)
├── templates/
│   ├── claudemd-template.md                     Section structure for generated CLAUDE.md
│   ├── question-bank.md                         Full question catalog with branching rules
│   ├── private-scaffold/                        Vendored from ai-pipeline-scaffold — full pipeline
│   │   ├── .claude/
│   │   │   ├── agents/                          8 pipeline agents
│   │   │   ├── skills/                          6 pipeline skills
│   │   │   ├── commands/improve.md
│   │   │   └── settings.example.json
│   │   ├── ARCHITECTURE.md
│   │   ├── CUSTOMIZE.md
│   │   ├── blocked-paths.txt
│   │   ├── .gitignore, .gitattributes
│   └── public-scaffold/                         Vendored — static-scope template for public side
│       ├── .claude/
│       │   ├── agents/worker.md
│       │   ├── skills/core-workflow/SKILL.md
│       │   ├── commands/run.md
│       │   └── settings.example.json
│       ├── CLAUDE.md, README.md, .gitignore
└── examples/
    └── example-claudemd.md                      Worked example of generator output
```

## How interview answers map to outputs

The reason the interview is a single skill instead of two: one answer feeds many files. Highlights:

| Answer | Lands in (private/single side) | Lands in (public side, dual only) |
|---|---|---|
| Project name | Every `{{PROJECT_NAME}}` placeholder | Same |
| "What this is NOT" / scope walls | `CLAUDE.md`, `architect.md` scope rules, `architect-review/SKILL.md` | `CLAUDE.md` only |
| Data sensitivity + blocked paths | `blocked-paths.txt`, `CLAUDE.md` data section | Empty `blocked-paths.txt` |
| Gotchas | `CLAUDE.md`, `debug/SKILL.md` failure modes, `implementer.md` landmines | `CLAUDE.md` |
| AI collaboration rules | `CLAUDE.md`, plus reinforcing `publisher.md` safety rails | `CLAUDE.md` |
| Pipeline opt-in | If `no`: prune `.claude/agents/`, `.claude/skills/`, `.claude/commands/improve.md` from private side | Public never has the pipeline regardless |

See `CLAUDE.md` for the full mapping table.

## Relationship to `ai-pipeline-scaffold`

Templates from `ai-pipeline-scaffold` (the 8 agents, 6 skills, public-scaffold template, blocked-paths format) are **vendored** under `templates/`. Going forward, this repo is the canonical bootstrap tool. `ai-pipeline-scaffold` remains a portfolio piece that documents the pipeline design philosophy.

When upstream `ai-pipeline-scaffold` evolves, sync changes manually:
1. Diff `C:\Projects\ai-pipeline-scaffold/.claude/agents/` against `templates/private-scaffold/.claude/agents/`.
2. Pull forward improvements that aren't bootstrap-specific.
3. Regenerate `examples/example-claudemd.md` if section structure changed.

`ai-pipeline-scaffold`'s `/begin` and `bootstrap-project` skill are deliberately NOT vendored — they're replaced by `/new-project` here.

## Design principles

- **Depth over breadth.** Better to ask 12 thoughtful questions than 30 shallow ones. The cap is a feature.
- **One source of truth per fact.** The interview captures it; the skills propagate it. The user shouldn't see the same question twice.
- **Specific over verbose.** Generated CLAUDE.md targets <200 lines. Per Anthropic's guidance, bloated CLAUDE.md causes Claude to ignore rules.
- **Preview before disk.** Both commands show the full assembled output before any write. The user always gets a chance to revise.
- **Fail loudly, refuse safely.** Targets exist? Refuse. Required answers missing? Stop. Existing files in the way? Back them up; never silently overwrite.
- **No remote, no commit.** This tool `git init`s. Staging, committing, and pushing are the user's call.

## License

MIT (matches `ai-pipeline-scaffold`).

## Author

Kyle Rauch — built in collaboration with Claude (Anthropic).
