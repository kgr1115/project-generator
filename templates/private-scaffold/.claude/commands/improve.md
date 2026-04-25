---
description: Run the improvement pipeline on this project. Invokes the orchestrator agent, which dispatches researcher → architect → implementer → tester → publisher (with debugger on failures). Use when you want systemic improvements rather than a one-off edit.
argument-hint: [optional focus area — e.g. "onboarding", "publisher safety rails" — omit for a full audit]
---

Focus area (if any): `$ARGUMENTS`

The user wants to run the improvement pipeline. Spawn the `orchestrator` subagent (`.claude/agents/orchestrator.md`) with the following brief:

- **Task**: Run one improvement cycle on this project.
- **Scope**: Read `CLAUDE.md` for the project's standing brief and scope rules. Read `.claude/agents/architect.md` (and `.claude/skills/architect-review/SKILL.md`) for the architect's scope gate. If either still contains placeholder scope rules, fall back to `CLAUDE.md`'s scope section.
- **Focus**: If `$ARGUMENTS` is non-empty, narrow the researcher's audit to that focus area. Otherwise, full audit.
- **Authorization envelope**: Honor any envelope the user stated in this session. **If the user did not state one**, the default envelope for `/improve` is:
  - Local commits: APPROVED.
  - `git push`: NOT approved (the orchestrator must ask before pushing).
  - Force-push, branch deletion, irreversible real-world actions, real user-data mutation: NEVER approved without explicit per-item confirmation.
  - Publisher safety-rail changes (personal-data guard, hooks, staging discipline): NOT approved.

  **If the user's intent is unclear**, the orchestrator must briefly confirm the envelope (one-line summary) with the user before spawning the researcher. Don't guess; don't proceed under an envelope wider than this default.

  Users who want to override the default (e.g. pre-approve pushes for the cycle) should state that in their `/improve` invocation or in the same message.

Then hand off to the orchestrator. Do NOT run the pipeline yourself in this command thread — the orchestrator is the coordinator.

If the `Task` tool / subagent spawning is not available in the current runtime, the orchestrator's profile includes a fallback-mode section describing how to execute the pipeline sequentially in-thread. Read that and proceed accordingly.

Return the orchestrator's final summary verbatim — the user reads that directly.
