---
name: debug
description: Invoked by the debugger agent to diagnose broken or unexpected behavior in the {{PROJECT_NAME}} pipeline. Maps each symptom class to its canonical evidence source, encodes known failure modes, and defines a decision tree for spawning sub-debuggers. Argument is a free-text symptom description.
argument-hint: <symptom description>
---

Symptom: `$ARGUMENTS`

---

## Step 0 — State the symptom precisely

Rephrase the incoming report as one sentence of observable behavior before touching any files.

- Bad: "the watcher seems broken"
- Good: "watcher.log shows 3 tick lines then stops; no ingests since 14:22"

---

## Step 1 — Map symptom to evidence source

> **NOTE TO ADOPTER:** Populate this table for your project. Each row maps a symptom class to the exact file/log/command that will tell you what's happening. Generic examples are shown; replace with project-specifics.

| Symptom class | Go-to evidence |
|---|---|
| Background process not running | Check PID file; tail the relevant log file; `ps` / `tasklist` for the expected process |
| Ingester errors / partial ingest | Tail the ingester's log; check the "processed" folder for archived payloads; inspect data-store state for partial writes |
| Artifact not reflecting underlying data | Regenerate the artifact manually and capture stderr; diff what's in the output vs what's in the source |
| Agent/skill runtime error | Capture full stderr from the script; check installed-package versions; `.claude/settings.json` permissions list |
| Permission / hook misfire | `.claude/settings.json` → `permissions.allow` list; check `hooks` block; `.claude/agents/*.md` for which agent runs what |
| Stuck / runaway process | Process list with age; file mtimes on expected outputs; cost signal if it's a paid-API loop |

---

## Step 2 — Known failure modes for this project

> **NOTE TO ADOPTER:** This is a TEMPLATE. Populate this section with the failure modes that have actually hit your project. Generic examples are given to show the format. Each entry should have:
> - A clear name/identifier (F1, F2, ...)
> - The observable symptom
> - The root cause
> - How to check (the diagnostic command)
> - How to fix, or a pointer to the canonical fix

### F1 (example). Cross-platform path handling

**Symptom:** A script that works on one platform fails on another with `FileNotFoundError`, `[WinError 17]`, or unexpected path components.
**Cause:** String concatenation of paths with `\` or `/` instead of using path-aware APIs. Cross-volume moves on Windows with non-atomic APIs.
**Check:** Grep the suspect file for string concatenation around path variables.
**Fix:** Use `pathlib.Path` (or the project's path-abstraction layer). For moves, prefer atomic APIs over copy+unlink sequences.

### F2 (example). Duplicate background-process instances

**Symptom:** Work appears to be done twice; duplicate log entries at slightly different timestamps; idempotency guards keep firing.
**Cause:** Multiple copies of a background process started (shell profile auto-start + manual start; stale PID lockfile from a crashed previous run).
**Check:** Process list for how many instances of the process are running. Check the PID lockfile's age and whether the PID it points to is still alive.
**Fix:** Kill the duplicate; clear the stale PID file; confirm only one instance starts on the next launch.

### F3. Template placeholder accidentally substituted in-place

**Symptom:** A scaffold file that should contain double-curly template tokens (for project name, end-user description, repo roots, etc.) now contains a concrete value, and the bootstrap tool (`bootstrap-project` skill) is producing spawned projects that carry the wrong project's name baked in.
**Cause:** An implementer treated the placeholder as an "unresolved variable that needs filling in" and substituted it. Placeholders are intentional — they're substituted by `bootstrap-project` when copying into a fresh spawned repo, not at author time.
**Check:** `git diff HEAD~1 HEAD -- .claude/agents/*.md .claude/skills/*/SKILL.md templates/public-scaffold/**` for any file where `{{...}}` tokens disappeared.
**Fix:** Revert the in-place substitution. If a scaffold-self file genuinely needs to refer to this repo's name, write `ai-pipeline-scaffold` as a literal (not a placeholder), and only in places where it's describing the scaffold itself rather than functioning as a template.

### F4. Publisher REFUSED — staged file matched personal-data blocked list

**Symptom:** Publisher returns `Status: REFUSED — personal data detected in staging area` even though the diff looked routine.
**Cause:** A path that looked innocuous (e.g. a generated artifact, a log file, a renamed config) matched a glob in the publisher's blocked-paths list. The list is project-specific and lives in `publisher.md` Step 3 / `publish-change/SKILL.md` Step 2 — adopters are supposed to populate it; sometimes a path the adopter forgot about gets caught.
**Check:** Read the implementer's declared file list and the publisher's blocked-paths list. Compare against `git diff --cached --name-only` output. The match is usually exact or via a glob pattern.
**Fix:** Either (a) the matched file genuinely shouldn't ship — `git restore --staged <file>` and re-run the publisher, or (b) the blocked-paths list is too broad — narrow it via a more specific glob, document the change, and re-run. Never bypass the guard.

### F5. Skill/agent profile fails the frontmatter validator

**Symptom:** Tester returns FAIL on Phase 1 with output from the Python validator block — typical messages: "name has bad chars", "description >1024 chars", "name X != Y" (name doesn't match filename or parent dir).
**Cause:** The recently-edited profile drifted from Anthropic's documented limits (`name`: ≤64 chars, lowercase-hyphen-digit only, not reserved, matches dir/filename; `description`: ≤1024 chars, non-empty). Common culprits: someone substituted a placeholder inside a `name:` field; a description got bloated past 1024; a skill directory was renamed without updating its SKILL.md `name`.
**Check:** Re-run the validator block from `tester.md` Phase 1 / `test-change/SKILL.md` Phase 1 — its error message names the exact rule violated.
**Fix:** Edit the `description:` to under 1024 chars (drop implementation details — that's body content, not routing signal); or rename `name:` to match the filename/dir; or rename the dir to match `name:`. After fix, re-run the validator until it returns `frontmatter OK`.

### F6. bootstrap-project Phase 3/4 mkdir fails on Windows path with spaces

**Symptom:** `bootstrap-project` skill aborts during Phase 3 (private repo creation) or Phase 4 (public repo creation) with a path-related error — `mkdir: cannot create directory ... No such file or directory`, or copy commands choke on unquoted spaces in `<parent-dir>`.
**Cause:** The parent-directory answer from Phase 1 contains spaces (e.g. `C:\Users\Kyle Rauch\projects\`), and the bash invocation didn't quote it. Bash on Windows treats unquoted spaces as argument separators.
**Check:** Look at the `parent-dir` value the user provided in Phase 1. If it contains spaces, examine the actual `mkdir` / `cp` invocation that failed — it's almost certainly missing quotes around the path variable.
**Fix:** Quote every shell expansion of `<PRIVATE_PATH>`, `<PUBLIC_PATH>`, and `<parent-dir>` in the commands the skill runs. As a workaround for an in-flight bootstrap, the user can re-answer Phase 1 with a parent dir that has no spaces (`C:\projects\`), or the implementer can patch the skill to wrap path variables in double-quotes throughout.

### F7. Phantom diffs from CRLF/LF line-ending drift on cross-platform git

**Symptom:** After committing on Windows, `git status` immediately shows the file as modified again — or every file in the repo appears "modified" after a fresh clone on a different OS, even though no one changed anything. CI/sister-clones see diffs that aren't in the original commit. May surface as `warning: in the working copy of '...', CRLF will be replaced by LF the next time Git touches it`.
**Cause:** No `.gitattributes` to normalize line endings. Windows clients write CRLF; Unix/macOS clients write LF; git's `core.autocrlf` setting is per-machine and inconsistent. Without a repo-level normalization rule, every cross-platform contributor reintroduces the drift.
**Check:** `git ls-files --eol` shows current vs. expected EOL per file. If you see mixed `i/` (working copy) values across text files that should be uniform, that's drift. Also: presence/absence of `.gitattributes` at repo root.
**Fix:** Add a `.gitattributes` at repo root with `* text=auto eol=lf` (or `eol=crlf` if the project genuinely needs Windows-native endings — uncommon). Commit it. On affected machines, run `git add --renormalize .` once to apply the rule retroactively, then commit the renormalization. The scaffold itself shipped this fix in commit `c01a5c4` after cycle-2 surfaced phantom diffs on cross-platform clones.

### F8, F9, ... Additional project-specific failure modes

> **NOTE TO ADOPTER:** Add your real failure modes here as they're discovered. Each one discovered-and-documented saves the next debugger a round trip.

---

## Step 3 — Gather evidence before forming hypotheses

For each symptom class, collect in this order:

1. **Logs first.** Tail the primary runtime log. Look for `ERROR`, `FAILED`, `crashed`, and timestamps that stop too early.
2. **File system state.** mtimes on output files tell you what completed and in what order.
3. **Data-store state.** Open the shared data store (spreadsheet/DB/JSON) and read the relevant rows. This is the canonical state — derived views (dashboards, reports) come from this.
4. **Process list.** For stuck subprocesses — especially any headless agent loops that could be burning tokens.
5. **Return codes.** If a script ran, find its exit code. Non-zero means it diagnosed its own failure — read stderr.

---

## Step 4 — Decision tree for spawning sub-debuggers

Spawn sub-debuggers when the issue decomposes into independent threads. Cap at 3 parallel sub-debuggers.

```
Is the failure in a SINGLE component (one script, one service, one agent)?
  YES → investigate directly. No sub-debugger needed.
  NO  → does it split into independent sub-investigations?
          YES → does each sub-investigation require different evidence sources?
                  YES → spawn one sub-debugger per sub-investigation
                  NO  → investigate the shared evidence source yourself first,
                         then spawn sub-debuggers for remaining unknowns
          NO  → investigate sequentially (one root cause often explains all symptoms)
```

**Real decomposition example:** "pipeline produced partial results" decomposes into:
- (a) Did the orchestrator process exit early? (process list + log)
- (b) Did any downstream step hang or produce a 0-byte output? (file mtimes + lockfiles)
- (c) Did the state-update step fail after the artifact was produced? (data-store state)

These are independent — spawn three sub-debuggers, merge results.

**Do NOT spawn a sub-debugger** to grep a log file or check a known failure mode from this skill. Those are 5-minute tasks.

---

## Step 5 — Safety rails (do not violate)

> **NOTE TO ADOPTER:** Customize the specific paths and processes listed here for your project.

- **Do NOT write to shared data stores** during a debug session. Read only. If a fix requires a data-store mutation, call it "Needs user's approval" and stop.
- **Do NOT move or delete files from live ingester input folders** to "test" the ingester. Use test fixtures in a separate folder.
- **Do NOT run ingesters against real payloads** without confirming (a) they haven't already been processed and (b) the data store doesn't already have the corresponding rows. Idempotency protects against data duplication but not against confusing the user.
- **Do NOT kill shared-desktop processes** (word processors, IDEs, browsers) that the user may have other state in. Flag to the user instead.
- **Do NOT kill a long-running agent process** unless it is visibly stuck (process age > 30 min with no file output change). Document PID and age before killing.
- **Do NOT touch read-only source material** for any reason.

---

## Step 6 — Output format

Every debug report ends with exactly this structure:

```markdown
## Root cause
{One paragraph. Name the mechanism, not just the symptom.}

## Evidence
- {file path / line / log timestamp / command output excerpt}
- ...

## Recommended fix
{Concrete change. Name the files. Note trade-offs.}

## Safety assessment
Safe to apply / Needs review / Needs user's approval — and why.

## Open questions
{Anything you couldn't fully answer. Empty is fine.}
```

---

## Non-negotiables

- Never invent a root cause without evidence. "I couldn't isolate this with available evidence; recommend instrumenting X and retrying after next repro" is a valid report.
- Never push to git. The orchestrator handles pushes.
- Never apply fixes that touch real user data, ingesters, or the data-store schema without the user's explicit approval.
