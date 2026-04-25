# project-generator — standing brief

> Read this first when working in this repository.

## What this project is

A unified **bootstrap-and-brief tool** that runs inside Claude Code (or another LLM coding tool). For any new or existing project, it interviews the user about purpose, stack, conventions, gotchas, and AI collaboration preferences, then either:

- **`/new-project`** — scaffolds a new project (single-repo OR paired private/public), optionally seeds the agent improvement pipeline, and writes a tailored `CLAUDE.md`. Replaces `ai-pipeline-scaffold/.claude/commands/begin.md`.
- **`/generate-claudemd`** — writes only a tailored `CLAUDE.md` into an existing target directory. For retrofitting projects that already have code but no useful brief.

Both commands share one interview skill so the question flow stays consistent.

## What this project is NOT

- Not a code-analysis tool. It does not parse a target codebase to infer everything (Claude Code's built-in `/init` does that). The value here is the **interview** — capturing intent, scope walls, anti-patterns, gotchas, and AI collaboration preferences that don't live in the code.
- Not a SaaS. No hosted services, no telemetry, no credentials.
- Not a runnable application. The "product" is the slash commands, the skills, the templates, and the vendored agent pipeline.

## Relationship to `ai-pipeline-scaffold`

`ai-pipeline-scaffold` (at `C:\Projects\ai-pipeline-scaffold`) is the conceptual ancestor. Its agent profiles, skill playbooks, and public-scaffold template are **vendored** into this repo under `templates/private-scaffold/` and `templates/public-scaffold/`. project-generator is now the canonical bootstrap tool; the vendored content is the source of truth for spawned projects going forward.

When the upstream scaffold evolves, sync changes manually:
1. Diff `C:\Projects\ai-pipeline-scaffold/.claude/agents/` against `templates/private-scaffold/.claude/agents/`.
2. Pull forward improvements that aren't bootstrap-specific.
3. Update the example output if section structure changes.

ai-pipeline-scaffold's `/begin` and `bootstrap-project` skill are deliberately NOT vendored — they're replaced by `/new-project` here.

## Primary entry points

| Command | When to use | What it does |
|---|---|---|
| `/new-project [name]` | Greenfield — starting a new project from nothing | Asks layout (single/dual), runs interview, copies `templates/private-scaffold/` (and `templates/public-scaffold/` if dual), substitutes placeholders, writes tailored `CLAUDE.md`, `git init`. |
| `/generate-claudemd [target]` | Retrofit — existing repo missing a useful CLAUDE.md | Pre-scans manifests, runs shorter interview, writes only `CLAUDE.md` into the target directory. No scaffolding. |

Both delegate the question flow to `.claude/skills/interview-project-intent/SKILL.md` so the wording stays consistent.

## How the bootstrap flow works (high level)

`/new-project`:

1. **Resolve name + parent dir.** Refuse if target already exists.
2. **Ask layout.** `single` (one repo) or `dual` (paired `<name>-private/` + `<name>-public/`). No default; user picks actively.
3. **Ask pipeline opt-in.** Include the improvement pipeline (researcher → architect → implementer → tester → publisher)? Default yes for dual; default ask for single.
4. **Interview** via `interview-project-intent` skill.
5. **Preview the plan** — paths, what gets copied, scope rules pulled from interview answers.
6. **Confirm** — `proceed? [y/N]`.
7. **Execute:**
   - Copy `templates/private-scaffold/` → `<name>-private/` (or `<name>/` for single).
   - If pipeline opted out, prune `.claude/agents/`, `.claude/skills/`, `.claude/commands/improve.md`, `ARCHITECTURE.md`, `CUSTOMIZE.md` from the destination — leave just `.claude/settings.example.json` and a stub `.claude/` dir.
   - If dual, copy `templates/public-scaffold/` → `<name>-public/`.
   - Substitute placeholders (`{{PROJECT_NAME}}`, `{{END_USER_DESCRIPTION}}`, `{{PUBLIC_REPO_ROOT}}`, etc.).
   - Replace placeholder scope rules in `architect.md` and `architect-review/SKILL.md` with the user's interview answers.
   - Seed `blocked-paths.txt` from data sensitivity answer.
   - Write tailored `CLAUDE.md` (using `templates/claudemd-template.md`) to private side; write a stripped public-side variant if dual.
   - `git init` in each created repo. No remote.
8. **Report** — paths, next steps, what got pruned.

`/generate-claudemd` is the tail of that flow — interview + assemble + preview + write — with no scaffolding.

## Output requirements (what a generated CLAUDE.md must look like)

From Anthropic's official guidance plus this project's interview design:

- **Target under 200 lines.** Bloated CLAUDE.md causes Claude to ignore rules. Cut aggressively.
- **Specific, verifiable instructions.** "Use 2-space indentation" beats "format code properly."
- **Markdown headers and bullets,** not paragraphs.
- **Include only things Claude can't infer from code.**
- **Exclude:** standard conventions, file-by-file descriptions, anything in `package.json` already, self-evident practices, long tutorials.
- **Use emphasis for hard rules:** `**IMPORTANT:**` or `**YOU MUST:**`.
- **Use `@imports` for living references:** `@README.md`, `@package.json` instead of restating their contents.
- **Smell test each line** with: *"Would removing this cause Claude to make mistakes?"* If no, cut it.

## The interview's design principles

- **One question at a time.** Walls of questions produce shallow answers.
- **Branch on answers.** TypeScript answers don't drag the user through Python questions; solo answers skip team workflow.
- **Default-first.** Most questions offer 2–4 plausible defaults via `AskUserQuestion`; free-text only when genuinely open-ended.
- **Capture the *why*** for non-obvious answers — that's what makes rules durable instead of cargo-culted.
- **Cap depth.** Greenfield: 12–15 questions. Retrofit: 6–10. The user's "stop" command always wins.

## How interview answers map to outputs

The interview's answers feed multiple files. This is the load-bearing reason the interview is shared between commands.

| Interview answer | Lands in (private side) | Lands in (public side) |
|---|---|---|
| Project name | `CLAUDE.md`, `README.md`, `architect.md` (`{{PROJECT_NAME}}`) | Same |
| One-line description | `CLAUDE.md` "What this is", `README.md` | Same |
| End-user description | `orchestrator.md`, `researcher.md` (`{{END_USER_DESCRIPTION}}`), `CLAUDE.md` | Same |
| "What this is NOT" / scope walls | `CLAUDE.md` scope section, `architect.md` scope rules, `architect-review/SKILL.md` scope rules | `CLAUDE.md` scope section (private-only entries omitted) |
| Data sensitivity / forbidden paths | `blocked-paths.txt`, `CLAUDE.md` data section | Empty `blocked-paths.txt` (public side ships clean) |
| Tech stack | `CLAUDE.md` tech section | Same (if user-facing code lives in both) |
| Run/test commands | `CLAUDE.md` "How to run it" | Same |
| Conventions | `CLAUDE.md` conventions, `implementer.md` Phase 4 landmines | `CLAUDE.md` conventions only |
| Workflow (branches, commits) | `CLAUDE.md` workflow | Same |
| AI collaboration rules | `CLAUDE.md` AI rules, plus reinforcing `publisher.md` safety rails | `CLAUDE.md` AI rules |
| Gotchas / landmines | `CLAUDE.md` gotchas, `debug/SKILL.md` Step 2, `implementer.md` Phase 4 | `CLAUDE.md` gotchas |
| Pipeline opt-in (yes/no) | If no: prune private-side `.claude/agents/`, `.claude/skills/`, `.claude/commands/improve.md` | Public side never has the pipeline regardless |

The shared interview is the single source of truth — no answer is ever asked twice.

## Working rules for changes inside THIS repo

- `.claude/skills/interview-project-intent/SKILL.md` is canonical for the question flow. Both `/new-project` and `/generate-claudemd` invoke it; do not fork the questions into either skill.
- `templates/private-scaffold/` and `templates/public-scaffold/` are vendored from `ai-pipeline-scaffold`. Edits made here can drift from the upstream — note any deliberate divergence in this CLAUDE.md.
- Templates are data. Branching logic lives in the skills, not in template files.
- `templates/claudemd-template.md` defines section structure for generated output. Sections marked `[optional]` get stripped if their slot is empty.
- `examples/example-claudemd.md` is a reference output for a fictional project. Regenerate after structural template changes.
- New interview questions go in `templates/question-bank.md` first (with rationale + branching rules), then get wired into the skill flow.

## Absolute constraints

1. **Never write to a target directory without preview + confirmation.** Both commands MUST show the assembled plan and the assembled `CLAUDE.md` before any disk write.
2. **Never overwrite an existing `CLAUDE.md`, agent file, or skill file without explicit confirmation.** Offer: replace (with `.bak` backup), append, or abort.
3. **Never proceed if target directory is non-empty for `/new-project`.** Refuse and ask the user to clean up or rename.
4. **Never invent project facts.** If the user hasn't said something and Claude can't read it from code, ask — don't fabricate.
5. **Never include secrets, credentials, or PII in the output.** If the interview surfaces a sensitive value, confirm explicitly before writing.
6. **Never `git add` or `git commit` from these skills.** `git init` only. The user decides what to stage.
7. **Never push.** No remote operations from any skill in this repo.
8. **Never run `--dangerously-skip-permissions`.** Use `.claude/settings.json` (or `settings.example.json` from the vendored template) for scoped permissions.

## Reference docs

- `README.md` — human-facing overview.
- `templates/claudemd-template.md` — section structure of generated CLAUDE.md.
- `templates/question-bank.md` — full question catalog with branching rules.
- `templates/private-scaffold/ARCHITECTURE.md` — vendored from ai-pipeline-scaffold; pipeline philosophy + handoff contracts.
- `templates/private-scaffold/CUSTOMIZE.md` — vendored; downstream customization checklist (largely automated by `/new-project`).
- `examples/example-claudemd.md` — worked example of generator output.
- `.claude/skills/interview-project-intent/SKILL.md` — the shared question flow.
- `.claude/skills/new-project/SKILL.md` — full bootstrap playbook.
- `.claude/skills/generate-claudemd/SKILL.md` — retrofit-only playbook.
