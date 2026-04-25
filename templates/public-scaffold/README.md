# {{PROJECT_NAME}}

{{PROJECT_DESCRIPTION}}

## Who this is for

{{END_USER_DESCRIPTION}}

## Quick-start

1. Clone this repo.
2. Install dependencies: `<fill in — e.g. pip install -r requirements.txt>`
3. Run: `<fill in — e.g. python main.py>`, or inside Claude Code, invoke `/run` (see `.claude/commands/run.md`).

## How it's organized

```
{{PROJECT_NAME}}/
├── CLAUDE.md                            Standing brief for Claude Code
├── README.md                            This file
├── .claude/
│   ├── agents/
│   │   └── worker.md                    Example domain agent — rename/replace for your project
│   ├── skills/
│   │   └── core-workflow/SKILL.md       Example workflow playbook — rename/replace
│   └── commands/
│       └── run.md                       Example slash command
└── <your project source>
```

## Philosophy

This project has a **static scope**. The `.claude/` directory holds agents and skills tailored to the project's runtime purpose — not a generic improvement pipeline. If you want to change scope (new features, new dependencies, new services), that's a conversation between you and the user, not an automated process.

Skills are the playbook library. Agents are the execution contexts that run them. Slash commands are thin wrappers that spawn an agent. Add new ones as your project grows; follow the patterns in the existing examples.

## License

<fill in>
