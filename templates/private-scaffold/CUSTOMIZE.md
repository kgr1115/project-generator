# Customizing the scaffold for your project

This guide walks through adapting the scaffold from generic template into a working `.claude/` directory for a specific project.

## 1. Placeholders to replace

The scaffold uses `{{double-curly}}` placeholders throughout. Do a find-replace across all files:

| Placeholder | What to replace with | Where it appears |
|---|---|---|
| `{{PROJECT_NAME}}` | Your project's name (e.g. "acme-crm", "my-research-bot") | Every agent + skill file |
| `{{END_USER_DESCRIPTION}}` | A short phrase describing who benefits — e.g. "solo founders running the CRM", "SRE teams", "anyone who forks this tool locally" | `orchestrator.md`, `researcher.md` |
| `{{PRIVATE_REPO_ROOT}}` | Absolute path to the private/working fork if you use paired forks (e.g. `C:\Projects\acme-crm-personal`). Delete references if single-fork. | `publish-change/SKILL.md` |
| `{{PUBLIC_REPO_ROOT}}` | Absolute path to the public/shipping repo (e.g. `C:\Projects\acme-crm`). If single-fork, use your repo root. | `publisher.md`, `publish-change/SKILL.md` |
| `{{PUBLIC_REPO_URL}}` | GitHub/GitLab URL of the public repo, if applicable | `publisher.md` (optional — omit if you don't want it referenced) |
| `{{AI_COAUTHOR_IDENTITY}}` | Your preferred co-author attribution, e.g. `Claude <noreply@anthropic.com>`. Delete the whole co-author block if you don't want one. | `publisher.md`, `publish-change/SKILL.md` |

Do the find-replace FIRST. Most other customization is easier after the placeholders are resolved.

## 2. Decide: single-fork or paired public/private?

The scaffold supports both patterns. Pick one.

### Single-fork (most projects)

You have one repo. Push-worthy changes go to `main`; local-only changes stay on your working tree. This is the simpler model.

Customizations:
- In `publisher.md` and `publish-change/SKILL.md`, delete the "paired-fork verification" sections entirely.
- In `architect.md` and `architect-review/SKILL.md`, delete the "fork hygiene" approval criterion and any "public | private-only | both" scope annotations.
- In `implementer.md` and `implement-change/SKILL.md`, delete the "fork mirroring" section.
- Remove any remaining references to `{{PRIVATE_REPO_ROOT}}`.

### Paired public/private fork

You have two working trees:
- **Public fork** — a git repository, pushed to a remote, intended for others to read/fork.
- **Private fork** — your working copy with real user data, NOT pushed anywhere. May or may not be a git repository.

This pattern is useful when:
- You want to develop improvements against real data but ship only the code to the public.
- You want a portfolio artifact that shows the project's architecture without leaking your real usage.
- You want to prevent "auto-improvement" agents from running on the public fork — the public stays static for other forkers.

Customizations:
- Keep all fork-hygiene and fork-mirroring sections.
- Populate `{{PRIVATE_REPO_ROOT}}` and `{{PUBLIC_REPO_ROOT}}`.
- In the architect's approval criteria, retain the "fork hygiene" rule: user-facing features land in both; maintainer-only automation stays private.
- Framework-level files (agent profiles, skills, generators, ingesters) must stay byte-identical between forks for mirrored files.
- The private fork holds real user data; the public fork must never contain any. The publisher's "personal data guard" enforces this.

## 3. Customize architect scope constraints

This is the most important customization step. The architect's approval criteria are what keep Claude from "helpfully" adding paid APIs or hosted infra behind your back.

Open **both** `.claude/agents/architect.md` and `.claude/skills/architect-review/SKILL.md`. In each, replace the placeholder scope rules with your project's real constraints.

### Checklist — answer each question, then translate to a scope rule

1. **Cost model.** What are you willing to pay for?
   - Free + local only? (e.g. the only budgeted cost is LLM API tokens)
   - Paid APIs allowed up to $N/month?
   - Any managed service is fine?
2. **Dependency channels.** How are dependencies installed?
   - `pip` only? `pip + npm`? Docker? Vendored binaries?
   - Any new dependency needs explicit user approval?
3. **Deploy target.** Where does this run?
   - Single user's laptop? Team server? Hosted (Vercel/AWS)?
   - What does "works" mean — a CLI invocation, a running daemon, a deployed web app?
4. **User model.** Who's using it?
   - Single user, single-tenant — no accounts/auth/multi-user concepts.
   - Small team — shared state, still local.
   - Public product — accounts, auth, telemetry all on the table.
5. **Data sensitivity.** What's off-limits?
   - Read-only source material (originals the user never wants mutated).
   - Live state files the user is actively using.
   - Secrets, credentials, API keys.
6. **Shippability.** What's in the `main`-branch artifact?
   - Everything? Framework-only (paired-fork split)?
   - Is the public artifact a portfolio piece, a usable product, or both?

Each answer becomes a scope rule. Example — for a free-local-only project:

> **`{{PROJECT_NAME}}` IS:**
> - Free, local-only. Only budgeted cost: LLM API tokens.
> - Runs on one user's laptop via `python` + `claude` CLI.
> - Single-user. Personal data on-disk.
>
> **`{{PROJECT_NAME}}` IS NOT:**
> - A SaaS product. No hosted services, cloud DBs, accounts, auth, telemetry.
> - A multi-user platform.

## 3.5. Set up `.claude/settings.json` with a scoped permissions list

The scaffold ships a `.claude/settings.example.json` (at the scaffold root and in `templates/public-scaffold/.claude/`) showing a minimal scoped `permissions.allow` list for the tools the pipeline actually uses. Copy it to `.claude/settings.json` and trim.

Why this matters: every agent profile and skill in the scaffold explicitly forbids `--dangerously-skip-permissions`. The replacement is a *scoped* allow list that fails loudly if the pipeline tries an unexpected tool, rather than silently bypassing safety. Headless sessions (`claude -p ...`) inherit this config.

Rules of thumb:
- Start narrow; expand as you observe legitimate denials.
- Keep a `deny` list for the most dangerous flags (`git push --force`, `--no-verify`, `--no-gpg-sign`) as belt-and-suspenders.
- `settings.json` should be gitignored if it carries user-specific paths; `settings.local.json` is the conventional name for purely-local overrides (already covered by the template `.gitignore`).

## 4. Customize the "personal data guard" — populate `blocked-paths.txt`

The publisher's personal-data guard is a deterministic recipe (defined in `.claude/agents/publisher.md` and `.claude/skills/publish-change/SKILL.md`). It reads its blocked-paths list from a single file at the project root: **`blocked-paths.txt`**.

You don't edit the guard logic — you edit the list.

Open `blocked-paths.txt` and add the paths or globs your project must never push to a public remote. Lines starting with `#` and blank lines are comments. Every other line is either:
- a **literal path** (no glob characters): exact match against `git diff --cached --name-only`, or
- a **glob** (contains `*` or `?`): shell-glob match against each cached path.

Typical entries:
- Read-only source material directories (e.g. `data/originals/*`, `input/source/*`).
- Directories of generated real user artifacts (e.g. `reports/sent/*`, `exports/*`).
- Config files with secrets (e.g. `contact.json`, `credentials.json`, `.env` — though `.gitignore` should also catch these).
- Logs with real content (e.g. `logs/*.md`).
- Research/notes directories tied to real targets.

When in doubt, list it. The guard is defense-in-depth; a false positive just means the publisher refuses to push and you unstage the file manually.

If `blocked-paths.txt` is missing or contains only comments/blank lines, the guard soft-passes with a warning. That's the correct behavior for projects (like the scaffold itself) that have no real user data on disk.

## 5. Add project-specific failure modes to the debug skill

Open `.claude/skills/debug/SKILL.md`. Find Step 2 ("Known failure modes for this project"). The scaffold ships with two generic examples (F1 cross-platform paths, F2 duplicate background processes). Replace or extend.

Each entry should have:
- **Name** — a stable identifier (F1, F2, ...).
- **Symptom** — observable behavior.
- **Cause** — the mechanism.
- **Check** — the exact diagnostic to run.
- **Fix** — the canonical fix, or a pointer to it.

Add a new entry every time you finish debugging a novel failure mode. These compound over time.

## 6. Add project-specific landmines to the implementer

Open `.claude/agents/implementer.md` (Phase 4) and `.claude/skills/implement-change/SKILL.md` (Phase 4). Populate the "Known platform landmines" table with rules for patterns that have already bitten your codebase.

Example entries from real projects:
- "Use `os.replace()` not `shutil.move()`" (Windows cross-volume issues)
- "Use `shutil.which('claude')` to find the `.cmd` shim" (Windows `.cmd` vs `.exe` discovery)
- "After docx→pdf conversion, verify `pdf.stat().st_size > 0` before deleting the `.docx`" (silent COM conversion failures)
- "Never concatenate paths with string `+`; use `pathlib.Path`"

Every landmine you document saves a future implementer from reintroducing the bug.

## 7. Add project-specific agents

The scaffold includes only generic pipeline agents. Your project likely needs domain-specific agents layered on top:

- **Scout agents** — pull data from sources you care about (job boards, RSS feeds, GitHub, etc.).
- **Worker agents** — do the project's core value-generating task (tailor a resume, summarize a meeting, generate a chart).
- **Reviewer agents** — domain-specific quality checks beyond what the generic tester catches (e.g. a `conversation-flow-reviewer` for a chatbot, a `claims-verifier` for content that must stay honest).

Where to put them: `.claude/agents/<name>.md`, alongside the scaffold agents. Write them with the same structure (frontmatter + role description + how they work + constraints + output format).

**Who spawns them?** The orchestrator. Add a short mention in `orchestrator.md`'s "Spawn vs do-it-yourself" section describing when to spawn each domain-specific agent.

## 8. Add project-specific slash commands

Slash commands live in `.claude/commands/*.md` (not included in the scaffold — that's a separate concept from skills). Add them for user-invocable workflows like `/scout`, `/tailor 7`, `/research Acme`, etc.

Slash commands are thin wrappers that spawn the relevant agent. Skills are the richer library of workflow playbooks the agents follow.

## 9. Tune pipeline ergonomics

Once you've run the pipeline a few times, you may want to adjust:

- **Parallelism caps.** Orchestrator defaults to 10 concurrent subagents per batch. If your rate limits are lower, reduce. If your project does a lot of embarrassingly-parallel work, raise.
- **Fast-path rules.** The orchestrator has a "fast path" for trivial changes (skip researcher + architect, go straight to implementer → tester → publisher). Expand the list of fast-path-eligible change types as patterns emerge.
- **Escalation thresholds.** Default is "two failed tests → escalate to user." If your debugger succeeds reliably on first attempt, you might tighten to one. If your tests are flaky, you might loosen to three.
- **Model defaults.** Each agent's frontmatter sets a default model. If you find one is routinely over- or under-powered for its typical task, adjust.

## 10. Test the scaffold end-to-end

After customization, do a dry run:

1. Invent a small-but-real improvement you want made. (Example: "make the log timestamps ISO-8601 instead of epoch.")
2. Ask the orchestrator: "Run the improvement pipeline for: {your improvement}."
3. Watch what each stage does. Confirm:
   - Researcher produces a proposal in the right format.
   - Architect approves or denies with clear rationale.
   - Implementer writes the diff and hands off to tester.
   - Tester runs the checks and returns PASS/FAIL.
   - On PASS, publisher commits + pushes (or refuses with a clear reason if it shouldn't).

If a stage produces sloppy output, the fix is almost always in that stage's skill file. Tighten the steps, add examples, clarify the output format.

## What you'll probably need to tune in the first week

- **Architect scope rules** — the researcher will test them by proposing things. Every denial-with-revision-guidance is data about what the rule should more precisely say.
- **Debug skill failure-mode table** — fills up fast as you actually run the pipeline.
- **Personal data guard list in the publisher** — add paths the first time you almost-leak one.
- **Domain-specific agents** — you'll miss at least one when you first write the scaffold. Add as needed.
