# Architecture

This document describes the pipeline design, the handoff contracts between stages, and the rationale behind the choices. Read this before modifying the scaffold — most of the constraints exist because the obvious alternatives have failure modes.

## The pipeline

```
researcher  →  architect  →  implementer  →  tester  →  publisher
                                                 │
                                                 ├─ (fail) → debugger → tester (retest)
                                                 │                        │
                                                 │                        ├─ (pass) → publisher
                                                 │                        └─ (fail twice) → orchestrator (escalation)
                                                 └─ (pass) → publisher
```

Six sequential stages plus a fail-debug-retest loop, coordinated by an orchestrator.

### Why sequential, not async

Each stage's output is the next stage's input. You can't architect what hasn't been researched; you can't implement what hasn't been approved; you can't test what hasn't been implemented. Sequentialism is not a limitation — it's how scope gates work.

Parallelism lives INSIDE stages (fan out multiple scouts at once, tailor multiple items in parallel), not between stages.

### Why a binary pass/fail at each stage

Handoffs need unambiguous signals. "Mostly good" is not a pipeline state — the publisher can't know whether to push "mostly good." The architect APPROVES or DENIES. The tester PASSes or FAILs. The publisher SUCCEEDs or REFUSEs.

Uncertainty collapses to FAIL / DENIED — that's the cheap failure mode (a round trip). The expensive failure mode is approving-a-scope-breach or shipping-a-bug, so default to no when uncertain.

## Handoff contracts

Each stage produces a structured output the next stage can consume without re-reading the whole history.

### Researcher → Architect

**Input:** Free-form user request or autonomous session-start trigger.
**Output:** Markdown document with up to 10 proposals in a fixed format:
- Title
- Category
- Why it matters for the user (user-facing impact)
- Concrete change (files, behaviors, no hand-waving)
- Sources / precedent
- Estimated scope
- Risk
- Priority (P0 / P1 / P2 / P3)

### Architect → Implementer (on APPROVED)

**Output per proposal:**
- Verdict: APPROVED
- Rationale (1–2 sentences)
- Scope annotations (file-level constraints, fork scope, non-negotiables)
- Testing requirements (named edge cases the tester MUST cover)

### Architect → Researcher (on DENIED)

**Output per proposal:**
- Verdict: DENIED
- Rationale
- Revision guidance (what to change for re-submission, or "not viable" with reason)

### Implementer → Tester

**Output:**
- List of files changed with one-line purpose each
- Fork mirroring status
- Local verification results
- How to test (specific end-to-end scenario + architect's required edge cases)
- Known risks

### Tester → Publisher (on PASS)

**Output:**
- `Test result: PASS`
- Static check results
- Dynamic check results
- Edge case results
- Files to stage (explicit list)
- Commit message draft

### Tester → Debugger (on FAIL)

**Output:**
- Failing check and exact error
- Expected vs. got
- The architect's original approval (so the debugger doesn't fix beyond scope)
- Fix safety request: "Apply only if trivially safe. Otherwise return a recommendation."

### Debugger → Tester (retest)

**Output:**
- Root cause (mechanism, not symptom)
- Evidence (files, lines, logs, command output)
- Recommended fix (or applied fix if trivially safe)
- Safety assessment (Safe / Needs review / Needs user's approval)
- Open questions

### Publisher → done

**Output:**
- Status: SUCCESS or REFUSED
- On SUCCESS: commit SHA, staged files, remote URL, byte-identical diff confirmation for mirrored files
- On REFUSED: which rule was violated, evidence, recommended next step

## The orchestrator model (CEO, not middle manager)

The orchestrator is deliberately thin. Its job is to:

1. **Dispatch** — kick off pipelines; spawn one-shot subagents; route handoffs when agents can't hand off directly.
2. **Escalate** — step in only when a defined escalation trigger fires.
3. **Make prescribed approvals** — decisions the user has standing-authorized.

### What the orchestrator does NOT do

- **Re-check every output of every stage.** That collapses delegation. The researcher's proposals are trusted; the architect's verdicts are trusted; the implementer's diff is trusted. The tester is the check; the orchestrator is not a second check.
- **Perform the work itself.** If the orchestrator is tempted to "just write the code," it's bypassing the pipeline. Delegate or ask the user — don't collapse the stages.
- **Take irreversible actions without user approval.** The user's pre-authorization list is explicit. Anything not on it, ask.

### Prescribed approvals (the CEO is pre-authorized for these)

- Killing a proposal after 2 architect denials on the same idea.
- Breaking ties on architect-borderline calls.
- Choosing stage-retry budgets (fund another debugger attempt, or escalate).
- Overriding model tier on a specific subagent spawn.
- Fast-pathing trivial changes (skip researcher + architect).
- (If the project uses paired forks) auto-publishing cleared changes to the public repo that meet all safety criteria.

### Escalation triggers (the CEO intervenes)

- Tester fails twice (original + post-debugger retest).
- Architect denies the same proposal twice with revision guidance.
- Implementer reports impossibility within scope constraints.
- Publisher refuses (personal data, missing PASS, etc.).
- User asks directly ("what's the pipeline doing?", "why is X stuck?").

If none of the above fire, the orchestrator stays out of the pipeline's way.

## Bootstrap: how projects get created dual-repo from day one

The scaffold's default "create a new project" flow produces two repos at once — private (with the pipeline) and public (static-scope scaffold, no pipeline). The mechanism:

- `.claude/commands/begin.md` — user-facing trigger (`/begin`, or natural-language phrases routed via the skill description).
- `.claude/skills/bootstrap-project/SKILL.md` — the actual playbook. Gathers the project's name / description / scope, previews the plan, then copies files and substitutes placeholders.
- `templates/public-scaffold/` — source of the static-scope scaffold that lands in `<name>-public/`.
- This repo's own `.claude/agents/` + `.claude/skills/` (minus `bootstrap-project` and `commands/begin.md`) — source of the improvement pipeline that lands in `<name>-private/`.

Key invariants **for spawned projects**:

- **The bootstrap tool is not copied to spawned projects.** `begin.md` and the `bootstrap-project` skill stay in the scaffold — a spawned project doesn't need to spawn more projects.
- **The static-scope template ships to spawned projects' public side.** It's the "leaf template" — forkable, self-contained, no meta-tooling.
- **`skill-writer` is private-only for spawned projects.** Reusable, but drags the self-improvement cultural pattern along with it. Excluding it from public keeps the boundary crisp on projects where the pipeline is maintainer meta-tooling.
- **Fresh git init, no remote.** Both sides start as clean local git histories. The user adds remotes explicitly when ready — prevents accidental early pushes of private data.

### The meta-tool exception

The rules above govern **projects spawned from the scaffold**. They do NOT govern the scaffold itself.

For projects that use the scaffold, the improvement pipeline is meta-tooling: useful to the maintainer, unwanted baggage for forkers. Stripping it on the public side is correct.

For ai-pipeline-scaffold itself, the improvement pipeline is the deliverable — a forker of the scaffold wants all eight agents, all seven skills, the bootstrap tool, and the templates. Stripping any of that would defeat the point.

So the scaffold's own public face includes everything; only *downstream* projects get the stripped public fork. BOOTSTRAP.md has the full matrix.

See `BOOTSTRAP.md` for the user-facing flow.

## Why a private/public split is ONE valid pattern, not mandatory

Some projects benefit from a paired public/private fork:

- **Private fork** — maintainer's working copy, holds real user data, auto-improvement tooling runs here.
- **Public fork** — shipped artifact, static for other forkers, clean of real data, serves as portfolio + template.

This is useful when:
- You want to develop against real data without leaking it.
- The project is a showcase of AI-collaboration technique as much as a product.
- You want to prevent other forkers from running auto-improvement machinery on their own installs.

But it's NOT mandatory. Single-fork projects are simpler and perfectly valid. The scaffold supports both modes — see `CUSTOMIZE.md` for how to configure.

Signals that paired forks are worth it:
- Real user data is a first-class concern (personal info, credentials, proprietary content).
- The project has different audiences for source code (devs, forkers) vs. the running artifact (you, maintainer).
- Auto-improvement agents would be dangerous to run on a random forker's install without their knowledge.

Signals that single-fork is fine:
- No real user data lives in the repo.
- Everyone who uses the project also wants the improvement machinery.
- The project is a library or tool, not a personal-data-holding application.

## Safety rails

These are absolute. The orchestrator, agents, and skills all enforce them. Violating one is a bug, not a tradeoff.

### Never push without a tester PASS verdict

The publisher refuses to commit or push if the tester's handoff doesn't literally contain `Test result: PASS`. "It looked fine" summaries don't count. The tester's binary verdict is the authorization.

### Never auto-push personal data

The publisher runs a personal-data guard before every push. The guard scans `git status` for paths the project has marked sensitive (source materials, user-data files, credentials, logs). Any hit → REFUSE, report to user.

This catches the "I accidentally staged a real file" class of incident. Belt-and-suspenders with `.gitignore`.

### Never `git add -A` or `git add .`

Publisher stages files explicitly by name from the tester's PASS report. The tester knows what should be staged; bulk staging is how unrelated files leak in.

### Never skip commit hooks

No `--no-verify`. No `--no-gpg-sign`. Hooks exist for a reason. If a hook fails, the publisher refuses and the orchestrator decides next steps.

### Never `--dangerously-skip-permissions`

In any subprocess spawn, any `claude -p` invocation, any headless session. Use scoped `permissions.allow` rules in `.claude/settings.json` instead. Permissions failing loudly is better than permissions being bypassed silently.

### Never mutate user data unilaterally

The debugger, implementer, and tester all have explicit rules: if a fix requires touching the user's real data (source materials, tracker state, etc.), it's classified as "Needs user's approval" and stops. The orchestrator escalates.

### Never loop the debug-retest cycle more than twice

Tester FAIL → debugger → retest. If the retest also FAILs, escalate to orchestrator, then to user. Unbounded loops burn tokens and rarely succeed where two targeted attempts fail.

## Cost model

Rough token budget per pipeline run, using Anthropic's tiers as reference (scale mentally for other providers):

| Stage | Model | Approx. context | Approx. cost per invocation |
|---|---|---|---|
| Researcher (with external web research) | Sonnet | 20–40k tokens | Moderate |
| Architect (per proposal) | Sonnet | 5–10k tokens | Low |
| Implementer | Opus | 15–30k tokens | High |
| Tester | Sonnet | 10–20k tokens | Moderate |
| Debugger (when triggered) | Opus | 15–40k tokens | High |
| Publisher | Haiku | 5k tokens | Very low |

A typical pipeline run (1 proposal, happy path, no debugging) is dominated by the implementer and the research stage. A debug-retest cycle roughly doubles the cost.

### Cadence recommendations

- **High-activity phase (first week after scaffold adoption):** daily improvement pipelines are fine. Fixes compound quickly.
- **Steady state:** weekly. Most sessions won't produce improvements worth shipping; the researcher will correctly say "nothing high-leverage this cycle" some weeks.
- **On-demand:** whenever a specific UX friction or bug has been annoying you, kick off a pipeline targeting it directly.

Don't run the pipeline on every Claude Code session. Most sessions should be using the project for its intended purpose, not improving the project.

## When to add a new agent vs. a new skill

**Skill** when: the workflow is a repeatable set of steps that executes deterministically (or near-deterministically) each time. Examples: "ingest a manifest file," "generate the weekly report," "draft a cover letter from a template." Skills are the library of playbooks.

**Agent** when: the workflow needs independent decision-making, its own context window, or fan-out. Examples: "research this company end-to-end," "tailor this resume for this specific role," "debug this failure."

**Rule of thumb:** If you'd write the workflow as a numbered list of steps Claude follows, it's a skill. If you'd spawn a fresh Claude instance to handle it with its own context and judgment, it's an agent.

**Skills can be invoked by agents.** A Tailor agent might invoke the `implement-change` skill for its core workflow and a domain-specific `write-cover-letter` skill for the writing step. Agents are execution contexts; skills are the playbooks those contexts run.

## Extension points

The scaffold is deliberately generic. When adapting:

- Add **domain-specific agents** for things the scaffold doesn't know about (scout, tailor, interview-prep, etc.). Put them in `.claude/agents/`.
- Add **domain-specific skills** for repeatable workflows (`/research`, `/tailor 7`, `/scout`). Put them in `.claude/skills/<name>/SKILL.md`.
- Add **slash commands** for user-invocable workflows. Put them in `.claude/commands/<name>.md` (this is a separate concept from skills — commands are thin spawn-this-agent wrappers).
- Add **project-specific failure modes** to the `debug` skill as you encounter them. Each one documented saves a future debugger a round trip.
- Extend the **orchestrator's "Spawn vs do-it-yourself" section** with project-specific heuristics once you've run the pipeline a few times and observed what works.

The generic pipeline is the frame. Your project fills the frame with domain-specific bones.
