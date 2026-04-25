---
name: implement-change
description: Execute an architect-approved improvement to the {{PROJECT_NAME}} codebase — write or edit the code, syntax-check it, mirror to paired repos if the architect flagged public scope, and produce a tester handoff report. Invoked after architect-review returns an APPROVED verdict. Does NOT run tests and does NOT commit.
argument-hint: <proposal title or "next" to pick the next approved proposal>
---

Proposal: `$ARGUMENTS`

---

## Inputs required before starting

1. Researcher's original proposal (what, why, which files)
2. Architect's APPROVED verdict + scope annotations (constraints, fork scope, testing requirements)
3. Current `git status` of the target repo (baseline for the diff)

If any of these are missing, ask the orchestrator before touching any files.

---

## Phase 1 — Restate the contract

Write one paragraph in plain English: what you are about to change and why. Confirm it matches the architect's approval word-for-word.

If your summary diverges from the approval, **stop** — you've drifted. Ask the architect to clarify before proceeding.

---

## Phase 1.5 — User-facing docs discipline

If the architect's approval flags this change as user-facing (new feature, modified flow, new/changed slash command, new/changed agent behavior that the end user would see), the diff MUST include docs updates.

If the project uses a paired public/private fork and the public repo is a portfolio piece / template for others, update both repos' user-facing docs honestly — describe what the user contributes (vision/scope, domain expertise, quality control, real data, final authorization) and what AI contributes (code, generated artifacts, test/debug execution, subagent orchestration). Don't overclaim AI. Don't underclaim the user.

Internal refactors, typo fixes, log cleanup → docs update NOT required. Flag as non-user-facing in your handoff.

If the docs update is required and missing, the tester will catch it and the publisher will refuse to push. Save the round-trip — include it in the original diff.

---

## Phase 2 — Plan the diff

List every file you will touch. One sentence per file explaining what changes. If the list exceeds ~5 files, the proposal is over-scoped — hand it back to the architect for decomposition.

---

## Phase 3 — Implement

**Style rules:**
- Edit surgically. Match the existing code style in each file you touch.
- Don't refactor beyond what the proposal requires.
- Don't add comments unless WHY is non-obvious.
- Don't add backwards-compat shims or dead code for "later."

**Fork mirroring (only if the project uses paired forks):**
- If the architect's annotation says "both repos": edit the private fork first, then apply the identical change to the public. The public copy must be byte-identical for changed files.
- If "private-only": touch only the private fork.
- If the project is single-fork, skip this section.

---

## Phase 4 — Platform-specific landmines (check before finishing)

> **NOTE TO ADOPTER:** This is a TEMPLATE. Populate with failure modes that have already hit your project. Generic + scaffold-specific examples are given to show the format. Extend this table as you discover new project-specific landmines.

| Check | Rule |
|---|---|
| Template placeholders (double-curly tokens like the one in this skill's description) | Never substitute these in-place when editing scaffold files, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`, or `templates/public-scaffold/**`. Bootstrap substitutes them downstream. Only substitute when copying into a fresh spawned repo. |
| Cross-platform text substitution | Never shell out to `sed` for placeholder or pattern substitution — quoting rules differ across cmd / pwsh / bash. Use the Edit tool's `replace_all: true` instead. |
| `--dangerously-skip-permissions` | Never add this to any subprocess spawn. Use scoped `permissions.allow` in `.claude/settings.json` (see `settings.example.json`). |
| Path separators on cross-platform code | Use `pathlib.Path` or equivalent path-aware APIs; don't concatenate with `\` or `/` strings. |
| Path separators inside `.md` skill / agent files | Use forward slashes (`scripts/helper.py`) in any path written to a Markdown skill or agent profile, even when authoring on Windows. Backslashes break on Unix consumers. (Anthropic skill-authoring best-practices.) |
| Adding an in-thread fallback section to a stage agent | When extending an agent with a "fallback mode" (à la orchestrator's cycle-1 addition, or tester+debugger's cycle-2 addition), keep the agent's `tools:` frontmatter unchanged — `Task` stays listed as the default-mode tool; fallback is the exception, not a replacement. Add the fallback as an additive section that preserves binary verdicts and the existing handoff contracts; do not rewrite normal-mode instructions. |
| Cross-platform line endings | Every new project (and any cross-platform repo) needs a `.gitattributes` with `* text=auto eol=lf` at the repo root. Without it, Windows contributors reintroduce CRLF on every touch, surfacing as phantom diffs immediately after commit. See debug skill F7 for the full failure mode + fix. |
| Multi-line commit messages on Windows | Never write `git commit -m "$(cat <<'EOF' ... EOF)"` heredocs in skill files or scripts that may run in pwsh / cmd — heredocs are bash/zsh-only and fail silently on Windows-native shells. Use multiple `-m` flags (one per paragraph) for short messages, or `git commit -F <tempfile>` (placing the tempfile in the project working dir, not `/tmp/`) for longer ones. |

---

## Phase 5 — Local verification

Run these checks yourself before handing to the tester:

- Language syntax check on every file you touched.
- If you changed a generator that produces an artifact (dashboard, report, schema), regenerate the artifact and grep the output for the expected change.
- If you changed a running process (daemon, watcher, server), DON'T restart it yet — that's the tester's call.
- If you changed slash commands or agent profiles: read the frontmatter and confirm it parses, `name` matches filename, and description is routing-specific (not generic).
- If you changed a utility function: call it directly with representative inputs and confirm output.

---

## Phase 6 — Tester handoff report

Produce this block and pass it (with the actual diff) to the tester:

```markdown
### Implementation for: {proposal title}
**Files changed:**
  - `path/to/file` — {one-line purpose}
  - `path/to/other` — {one-line purpose}
**Fork mirroring:** both repos | private-only | N/A
  - {If both: diff confirmation that files are byte-identical}
**Local verification:**
  - syntax check: PASS | FAIL
  - artifact regen: PASS | N/A
  - {other checks}
**How to test (for the tester):**
  - {specific end-to-end user scenario}
  - {edge cases from architect's testing requirements}
**Known risks:**
  - {anything that could fail in testing that isn't obvious}
```

---

## Hard stops — when to escalate instead of proceeding

- Architect's annotation is ambiguous → ask the architect, don't guess.
- Implementation reveals the proposal needs more files than planned → hand back to architect for re-scoping.
- A change would touch an interface other modules depend on → pause; confirm blast radius was considered.
- The proposal turns out to require a dependency or service the project doesn't allow → flag to architect; don't substitute silently.

---

## Non-negotiables

- Never commit or push. Your work ends at a working diff + handoff report.
- Never touch user data: honor the standing brief's read-only list.
- Never expand scope silently. If a decision point makes the proposal bigger, escalate.
