---
name: generate-claudemd
description: Generate a tailored CLAUDE.md for an EXISTING project (retrofit). For new-project bootstrapping, use the new-project skill instead — that one handles directory creation, optional pipeline scaffolding, and dual/single layout. This skill writes only CLAUDE.md, into a target directory the user already has. Triggered by /generate-claudemd or natural-language phrases like "generate a CLAUDE.md for this repo", "set up Claude memory for this project".
argument-hint: [optional target directory — e.g. "../my-existing-repo" — omit to target cwd]
---

Input: `$ARGUMENTS` (target directory; defaults to `.`)

---

## When to use this skill

Use when the user has an EXISTING project and wants a real CLAUDE.md added or replaced. The target directory already has source code, manifests, etc. — this skill won't scaffold an agent pipeline or create directories.

**Don't** use this skill for:
- New project bootstrapping → use `new-project` instead.
- Editing an existing well-formed CLAUDE.md → just edit it.
- Pure code-discovery onboarding → Claude Code's built-in `/init` is better. This skill complements `/init` by capturing intent that code-reading can't.

---

## Phase 0 — Resolve target & detect mode

1. **Resolve target directory.**
   - If `$ARGUMENTS` is empty or `.`, target = current working directory.
   - Otherwise, target = `$ARGUMENTS` (absolute or relative).
   - If target doesn't exist, stop and ask the user to create it or supply a different path.

2. **Check for existing CLAUDE.md.**
   - If `<target>/CLAUDE.md` exists, read it. Tell the user the line count and ask: **append, replace (with `.bak` backup), or abort.** If abort, stop.

3. **Detect mode.**
   - **Greenfield** (rare for this skill — usually means user wants new-project): target is empty or contains only `.git/`, `.gitignore`, `LICENSE`, short `README.md`. If detected, prompt: "This directory looks empty — did you mean `/new-project` for a full bootstrap? [continue with /generate-claudemd | switch to /new-project | abort]".
   - **Retrofit:** target contains source code or manifests. Proceed with pre-scan.

4. **Pre-scan manifests** (retrofit only — capped at ~10 file reads to keep context lean):
   - Look for and read (if present): `package.json`, `pnpm-lock.yaml`, `package-lock.json`, `bun.lockb`, `pyproject.toml`, `requirements.txt`, `Cargo.toml`, `go.mod`, `Gemfile`, `*.csproj`, `pom.xml`, top-level `README.md`.
   - Extract inferences:
     - Primary language (from manifest type)
     - Package manager (from lockfile)
     - Test runner (from `package.json` scripts or pyproject.toml dependencies)
     - Run command (from `package.json` scripts.dev/scripts.start, or `Procfile`)
     - Framework hints (e.g. `next`, `fastapi`, `gin` in dependencies)
     - Project description (from `package.json.description` or `README.md` first line)
   - **DO NOT read source code files.** Pre-scan is manifest-only. Deeper inference is `/init`'s job.

5. **Announce the plan.** "I'll ask roughly 6–10 questions, one at a time. I've pre-filled some answers from your manifests — you'll just confirm those. Say 'skip' to any optional question or 'stop' to generate from what we have." Then proceed to Phase 1.

---

## Phase 1 — Run the interview

Invoke the `interview-project-intent` skill with mode = `retrofit`. Pass the pre-scan inferences from Phase 0 step 4 so the interview confirms rather than re-asks them.

The interview returns a structured answer dict (see `interview-project-intent/SKILL.md` Phase 3 for the shape).

If the interview returns `status: aborted` with too few answers (e.g. Q1, Q2, Q4 missing), tell the user the skill needs at minimum: project name (Q1), description (Q2), scope walls (Q4), and ask if they want to provide them now or abort. If they still abort, exit cleanly with no disk write.

---

## Phase 2 — Assemble

1. **Load the template** from `templates/claudemd-template.md`.

2. **Fill placeholders** from interview answers.

3. **Strip empty/optional sections.** A short CLAUDE.md is a feature.

4. **Apply formatting rules:**
   - Markdown headers and bullets only. No paragraphs over 3 lines.
   - Bold critical rules with `**IMPORTANT:**` or `**YOU MUST:**`.
   - Use `` `code` `` for every command, file path, and identifier.
   - Use `@README.md` / `@package.json` imports rather than restating their contents.
   - Use forward slashes in file paths (portable across OSes).

5. **Length check:**
   - Under 150 lines: ship.
   - 150–200: acceptable, do one trim pass.
   - Over 200: trim aggressively before showing the user. Cut convention details Claude already knows from the language; cut redundant phrasings; leave gotchas (Tier 8) intact — they're the highest-value content.

6. **Smell-test pass — for each line, ask:** *"Would removing this cause Claude to make a mistake?"* If clearly no, delete.

---

## Phase 3 — Preview & confirm

1. **Show the user the full assembled CLAUDE.md** in chat (no file write yet).
2. **Show summary stats:** line count, sections included, sections stripped, key answers used.
3. **Ask for revisions.** "Want any sections trimmed, expanded, or reworded? Or ready to write to disk?"
4. **Accept iterative revisions in conversation.** Common revisions:
   - "Remove the workflow section, I work alone."
   - "Add: never modify files in `data/originals/`."
   - "The gotcha about timezones isn't right — say [...]."
5. **Loop until the user approves.** No disk write until then.

---

## Phase 4 — Write

1. **Confirm target path one more time.** "Will write to: `<target>/CLAUDE.md`."
2. **If an existing CLAUDE.md was found in Phase 0:**
   - **Replace mode:** rename existing to `CLAUDE.md.bak` (warn if `CLAUDE.md.bak` already exists — the user has bootstrapped twice; ask before clobbering).
   - **Append mode:** read existing, write `<existing>\n\n---\n\n<new content>` to disk.
3. **Write the file** using the Write tool.
4. **Report:** "Wrote `<path>/CLAUDE.md` — N lines. Backed up previous to `CLAUDE.md.bak`." (omit backup line if not applicable)

---

## Phase 5 — Suggest follow-ups

Based on what was captured, suggest concrete next actions. The user may or may not act on them.

- **If forbidden actions captured (Q20):** "Consider creating `.claude/settings.json` with a scoped `permissions.allow` list to back the rules in CLAUDE.md. The vendored `templates/private-scaffold/.claude/settings.example.json` (in project-generator) is a starting point."
- **If gotchas captured (Q21):** "The gotchas you described are good candidates for hooks. The most enforceable ones (e.g. 'never use `shutil.move()`') can be turned into PostToolUse hooks that block specific patterns."
- **If the project has no `.claude/agents/` and the user mentioned wanting agent-style automation:** "If you want the agent improvement pipeline (researcher → architect → implementer → tester → publisher), `/new-project` (in project-generator) can scaffold that for a fresh repo. For an existing repo, manually copy `templates/private-scaffold/.claude/agents/` and `.claude/skills/` over."
- **If the user said `dual` layout in passing but this is retrofit:** "Heads up — `/generate-claudemd` doesn't create paired forks. If you want that structure, `/new-project` is the tool."

---

## Phase 6 — Report

Return a structured summary:

```
Skill: generate-claudemd
Status: SUCCESS | ABORTED | FAILED

On SUCCESS:
- Target: <path>/CLAUDE.md
- Mode: greenfield | retrofit
- Questions asked: N (cap was M)
- Pre-scan inferences confirmed: <count>
- Output length: N lines
- Sections included: <list>
- Sections stripped: <list>
- Backup created: <path>/CLAUDE.md.bak  (if applicable)
- Suggested follow-ups: <list>

On ABORTED:
- Reason: user stopped | existing CLAUDE.md, user chose abort | target inaccessible | minimum answers missing
- What was captured: <partial answers, if any>

On FAILED:
- Phase: <Resolve | Pre-scan | Interview | Assemble | Preview | Write>
- Error: <exact issue>
- Recommended next step: <e.g. "rerun in <new-target-path>", "manually create the dir first">
```

---

## Project-specific landmines

- **Don't read source code files in the pre-scan.** It's tempting to grep `src/` for context, but it bloats fast and duplicates `/init`. Manifest-level only.
- **Don't ask the user to opt in to the preview.** Just render the assembled markdown in chat.
- **`AskUserQuestion` returns one answer per call.** For multi-select scenarios in the shared interview (Q20), follow the catalog's instruction to ask sequentially or use free-text.
- **Path handling on Windows.** Use forward slashes in the markdown output even when the user is on Windows — `CLAUDE.md` content is portable; backslashes break tools that consume it on other OSes. Internal Bash invocations may use platform-native paths.
- **Don't commit the generated CLAUDE.md.** This skill writes the file; the user decides whether to stage and commit. Never invoke git from this skill.
- **If the existing CLAUDE.md is short (<30 lines) and the user picked `replace`, also consider whether they wanted `append`.** Offer once: "The existing CLAUDE.md is short — append might preserve context that took effort. Confirm replace?"
- **Do not silently write `CLAUDE.md.bak`.** Tell the user the backup path so they can find it if they later regret the replace.
