---
name: new-project
description: Full project bootstrap — interview the user, create a single-repo or paired private/public layout, optionally seed the agent improvement pipeline, write a tailored CLAUDE.md per side, git init. Replaces ai-pipeline-scaffold's /begin. Triggered by /new-project or natural-language phrases like "start a new project", "let's begin", "bootstrap a new repo".
argument-hint: [optional project name — e.g. "acme-crm" — omit to be prompted]
---

Input: `$ARGUMENTS` (project name; optional)

---

## When to use this skill

When the user wants to start a NEW project from nothing. Invoked by `/new-project`. For retrofitting an existing repo with a CLAUDE.md only, use `generate-claudemd` instead — that skill writes only `CLAUDE.md` and doesn't scaffold directories.

---

## Phase 0 — Preconditions

1. **Locate the templates.** Confirm `templates/private-scaffold/.claude/agents/orchestrator.md` and `templates/public-scaffold/CLAUDE.md` both exist relative to this skill's repo root. If either is missing, stop with an error: "templates are missing — repo may be corrupted or this skill is being run outside project-generator."

2. **Resolve project name.**
   - If `$ARGUMENTS` is non-empty, use it.
   - Otherwise, ask: "What's the project's name? (used for the directory and as `{{PROJECT_NAME}}`)"
   - Validate: lowercase, hyphens-or-underscores OK, no spaces, no special chars beyond `-_`. If invalid, ask again with a corrected suggestion.

3. **Resolve parent directory.**
   - Default: `C:\Projects\` (sibling of project-generator on Windows) or the user's typical projects root.
   - Confirm with the user: "Create the project as a sibling of project-generator, under `C:\Projects\`? Or a different parent directory?"
   - If the user supplies a path, validate it exists and is writable.

4. **Check for collisions.**
   - For dual layout (will be asked in Phase 1): if `<parent>/<name>-private/` OR `<parent>/<name>-public/` exists and is non-empty → STOP, ask the user to rename or clean up.
   - For single layout: if `<parent>/<name>/` exists and is non-empty → STOP.
   - (Defer the actual collision check until after Phase 1 when layout is known.)

---

## Phase 1 — Layout & pipeline questions

These two questions are bootstrap-only and don't fit in the shared interview cap. Ask them first, before the interview.

1. **Layout (Q6 in question bank).** Use `AskUserQuestion`:
   - "Repo layout? `dual` creates paired `<name>-private/` (working copy with real data) and `<name>-public/` (clean, shippable). `single` creates one repo. Pick `single` if the project never publishes or has no real-data concern."
   - Options: `dual`, `single`. **No default.** The user must actively choose.

2. **Pipeline opt-in (Q7).** Use `AskUserQuestion`:
   - "Include the agent improvement pipeline (researcher → architect → implementer → tester → publisher → debugger)? It's powerful but adds 8 agent files + 6 skills to the private side. Skip for one-shot scripts or trivial repos."
   - Options: `yes` (default), `no`.

3. **AI co-author (Q8).** Only if pipeline = `yes`:
   - "AI co-author attribution for commits the publisher makes?"
   - Options: `claude` (`Claude <noreply@anthropic.com>`, default), `none`, `custom` (free-text follow-up).

4. **Re-check collisions** now that layout is known. If collision, stop with a clear message.

---

## Phase 2 — Run the project-intent interview

Invoke the `interview-project-intent` skill with mode = `greenfield`.

Pass already-captured answers from Phase 1 (Q6, Q7, Q8) so the interview doesn't re-ask them.

The interview returns a structured answer dict (see `interview-project-intent/SKILL.md` Phase 3 for shape). Hold onto it.

If the interview returns `status: aborted`, ask the user: "Continue scaffolding with partial answers, or stop entirely?" If continue, proceed; if stop, exit cleanly with no disk writes.

---

## Phase 3 — Plan preview

Before any disk write, present the user with the full plan:

```
About to create:

  <parent>/<name>-private/   [dual layout]   ← full pipeline + tailored CLAUDE.md
  <parent>/<name>-public/    [dual layout]   ← static-scope scaffold + public-side CLAUDE.md

  OR

  <parent>/<name>/           [single layout] ← full pipeline + tailored CLAUDE.md

What gets copied to <name>-private (or <name>):
  ✓ .claude/agents/ (8 files)              [if pipeline opted-in]
  ✓ .claude/skills/ (6 skills)              [if pipeline opted-in]
  ✓ .claude/commands/improve.md             [if pipeline opted-in]
  ✓ .claude/settings.example.json
  ✓ blocked-paths.txt (seeded from data answers)
  ✓ ARCHITECTURE.md                         [if pipeline opted-in]
  ✓ CUSTOMIZE.md                            [if pipeline opted-in]
  ✓ CLAUDE.md                                [tailored from your interview]
  ✓ README.md                                [tailored stub]
  ✓ .gitignore, .gitattributes

What gets copied to <name>-public (dual layout only):
  ✓ .claude/agents/worker.md (placeholder example)
  ✓ .claude/skills/core-workflow/SKILL.md (placeholder example)
  ✓ .claude/commands/run.md
  ✓ .claude/settings.example.json
  ✓ CLAUDE.md  [public-side variant — strips private-only rules]
  ✓ README.md
  ✓ .gitignore

Scope rules (pulled from your interview Q4):
  <bulleted list of "this is NOT" answers>

Forbidden paths (seeded into blocked-paths.txt from Q5/Q5a):
  <bulleted list>

AI collaboration rules (from Q19, Q20):
  <bulleted list>

Pipeline status: <included | excluded>
git init: yes (no remote will be added)

Proceed? [y/N]
```

Wait for `y` (or "yes", "go", "proceed"). Anything else → abort cleanly.

---

## Phase 4 — Execute (private side)

For each created repo, do the following sequentially. Start with the private side.

### 4a. Create the directory and copy templates

1. `mkdir -p <parent>/<name>-private/` (or `<parent>/<name>/` for single)
2. Recursive copy from `templates/private-scaffold/` to the destination, preserving structure.

### 4b. Apply pipeline opt-out (if Q7 = no)

If pipeline was excluded:
1. Remove `<dest>/.claude/agents/` (entire directory)
2. Remove `<dest>/.claude/skills/` (entire directory)
3. Remove `<dest>/.claude/commands/improve.md`
4. Remove `<dest>/ARCHITECTURE.md`
5. Remove `<dest>/CUSTOMIZE.md`
6. Remove `<dest>/blocked-paths.txt` (no publisher to consume it)

The destination ends up with just `.claude/settings.example.json`, `.gitignore`, `.gitattributes`, `CLAUDE.md` (written next), and `README.md` (written next).

### 4c. Substitute placeholders

For every file under `<dest>/.claude/` and `<dest>/*.md` EXCEPT `CUSTOMIZE.md` and `blocked-paths.txt` (those are documentation that explains placeholders themselves — substitution would corrupt them), do find-replace:

| Placeholder | Replace with | Source |
|---|---|---|
| `{{PROJECT_NAME}}` | Q1 answer | interview |
| `{{END_USER_DESCRIPTION}}` | Q3 answer (use `Q3_end_user_other` if Q3 = `other`) | interview |
| `{{PRIVATE_REPO_ROOT}}` | absolute path of `<dest>` | computed |
| `{{PUBLIC_REPO_ROOT}}` | absolute path of public side (or same as private for single layout) | computed |
| `{{PUBLIC_REPO_URL}}` | `<unset — add when remote configured>` | placeholder; user fills later |
| `{{AI_COAUTHOR_IDENTITY}}` | Q8 answer (or empty if `none`) | interview |
| `{{PROJECT_DESCRIPTION}}` | Q2 answer | interview |

Use forward slashes in path substitutions even on Windows — the paths land inside markdown that's read by tools running on multiple platforms.

For single-layout projects, also strip the dual-fork sections per `CUSTOMIZE.md` §2 (paragraphs about "fork hygiene", "fork mirroring"). The vendored files do NOT yet have sentinel comment markers around these sections — if you find sentinels (`<!-- BEGIN PAIRED-FORK -->` / `<!-- END PAIRED-FORK -->`), use them; otherwise leave the dual-fork text in place and add a line to the post-bootstrap report flagging the user to manually trim. (Adding sentinels upstream is a tracked follow-up.)

### 4d. Strip-and-tidy vendored docs

The vendored templates contain "NOTE TO ADOPTER" blockquotes (template authoring notes addressed to the person customizing the scaffold) and "If this IS the ai-pipeline-scaffold repo itself" exception notes (scaffold meta-tool reminders). Both are out of place in spawned projects — strip them.

Apply to ALL `.md` files under `<dest>/` (including `<dest>/.claude/agents/*.md` and `<dest>/.claude/skills/*/SKILL.md`), EXCLUDING `CUSTOMIZE.md` (deleted next phase) and `blocked-paths.txt` (already special-cased).

Recipe — one perl pass per file:

```bash
# Strip blockquote ranges starting at NOTE TO ADOPTER or ai-pipeline-scaffold-itself markers.
# Each range ends at the first bare blank line (no leading `>`).
perl -i -ne '
  BEGIN { $skip = 0 }
  if (/^> \*\*NOTE TO ADOPTER/ || /^> \*\*If this IS the ai-pipeline-scaffold repo itself/) {
    $skip = 1
  }
  if ($skip) {
    if (/^\s*$/) { $skip = 0; print "\n"; next }
    next
  }
  print
' "$f"

# Then collapse runs of 2+ blank lines to 1, in-place.
perl -i -e 'local $/; my $c = <>; $c =~ s/\n{3,}/\n\n/g; print $c' "$f"
```

Verify: `grep -rn 'NOTE TO ADOPTER\|ai-pipeline-scaffold repo itself' <dest>` should return nothing (excluding `CUSTOMIZE.md`).

Files known to contain these blocks (as of the current vendored set; recheck if upstream changes): `architect.md`, `implementer.md`, `architect-review/SKILL.md`, `implement-change/SKILL.md`, `publish-change/SKILL.md`, `debug/SKILL.md`. Do not hard-code that list — find them via grep at runtime, since upstream may add or remove.

### 4e. Replace placeholder scope rules in architect files

Apply only if pipeline opted-in (Q7 = yes). Open `<dest>/.claude/agents/architect.md` and `<dest>/.claude/skills/architect-review/SKILL.md`.

After 4c substitution and 4d stripping, the scope blocks have one of two formats (the upstream files are inconsistent — handle both):

- **architect.md format:** `**IS:**` ... `**IS NOT:**` (no project name prefix)
- **architect-review/SKILL.md format:** `**<project-name> IS:**` ... `**<project-name> IS NOT:**` (project name prefix, post-substitution)

In each file, locate the placeholder bullet lines beginning with `- {{...}}` and replace the entire `**IS:**` ... `**IS NOT:**` block (including all `- {{...}}` bullets between) with content built from interview answers:

```markdown
**<project-name> IS:**            (or just **IS:** for architect.md — match the file's existing prefix)
- <Q2 description, declarative form>
- <derived from Q9 + Q10: language + runtime>
- <cost model derived from Q4 if any of its bullets mention cost>
- <layout: paired public/private fork — public side ships clean | single repo, local-only>

**<project-name> IS NOT:**
- <each Q4 bullet, verbatim>
```

If the user's Q4 answer is short (1-2 bullets), append the default scaffold rules (no paid services beyond what was approved; no SaaS / multi-tenant / hosted infra unless Q4 explicitly allows; no telemetry if Q3 = solo or small_team). Don't leave the file with empty scope.

### 4f. Delete CUSTOMIZE.md

`CUSTOMIZE.md` is the manual customization guide — but the bootstrap automated everything in it, AND it references scaffold-internal paths (`templates/public-scaffold/`, `bootstrap-project skill`) that don't exist in spawned projects.

Delete `<dest>/CUSTOMIZE.md` if present (no-op if pipeline=no, because 4b already removed it). It was never copied to the public side — see 5a.

If the user might want it as historical reference, move it to `<dest>/.dropped/CUSTOMIZE.md.original` instead — but default is delete. The bootstrap report should mention it was removed so the user isn't confused if they go looking.

### 4g. Seed blocked-paths.txt and/or .gitignore

The destination of the user's `Q5a` blocked-path entries depends on whether the pipeline is included. Without the pipeline, there's no publisher running the data guard at push time — so `.gitignore` becomes the only static defense.

**If pipeline = yes (Q7 = yes):**
- `<dest>/blocked-paths.txt` exists (vendored, kept by 4b).
- If Q5 ≠ `none` AND Q5a was answered:
  1. Open `<dest>/blocked-paths.txt`.
  2. Replace its content (or append after a `# user-supplied entries below` separator).
  3. Each Q5a entry → one line.
  4. Include comment headers explaining the format (`# literal paths or globs; one per line; lines starting with # are comments`).
- If Q5 = `none`: leave the file with comments only. Publisher's guard soft-passes on empty.

**If pipeline = no (Q7 = no):**
- `blocked-paths.txt` was removed by 4b. There's no publisher to consume it; nothing reads the file.
- If Q5 ∈ `{personal_local, customer_pii, regulated, secrets_only}` AND Q5a was answered, append Q5a paths to `<dest>/.gitignore` so git itself enforces them. The vendored `.gitignore` becomes the only static defense in pipeline=no mode.
- Append format (preserve existing `.gitignore` content; add a new section at the bottom):
  ```
  
  # Blocked paths (sensitive data — auto-added by /new-project)
  <Q5a entry 1>
  <Q5a entry 2>
  ...
  ```
- If Q5 = `none`: no action.

The bootstrap report should mention which file got the entries — users tend to assume `blocked-paths.txt` exists; explicitly noting "added to `.gitignore` because pipeline was excluded" prevents confusion.

### 4h. Write tailored CLAUDE.md

Use `templates/claudemd-template.md` to assemble. Fill placeholders from interview answers. Strip empty/optional sections. Apply the formatting rules from this skill's parent CLAUDE.md (the project-generator one): under 200 lines, specific, bulleted, `**IMPORTANT:**` for hard rules.

Write to `<dest>/CLAUDE.md`. (No prior CLAUDE.md exists on the private side — the vendored `templates/private-scaffold/` does NOT ship one; this is a fresh write.)

### 4i. Write tailored README.md

Generate the README freshly from interview answers. Do NOT substitute the public-scaffold's `<fill in>` placeholders — those are written for human-driven customization, and we have answers.

Structure:
- `# <project-name>` (Q1)
- One-sentence description (Q2)
- "Who this is for" — Q3 answer
- Quick-start block with verbatim install + run commands (Q14, Q13)
- "What's here" pointing at CLAUDE.md, .claude/, etc. — only the directories that actually got copied
- "Improvement pipeline" mention if Q7 = yes; omit otherwise
- "License" — leave as `MIT` if Q3 ∈ {oss, customers}, else `<fill in>` for the user to decide

Keep it under 40 lines. The user will rewrite this; the goal is a useful starting stub, not a polished doc.

### 4j. git init (private side)

`git init -b main` in `<dest>`. No remote, no first commit. The user decides what to stage.

(The `-b main` flag standardizes the default branch — older git defaults to `master`, newer git's default depends on user config. Being explicit avoids surprises.)

---

## Phase 5 — Execute (public side, dual layout only)

If layout = `single`, skip this phase entirely.

### 5a. Create directory and copy templates

1. `mkdir -p <parent>/<name>-public/`
2. Recursive copy from `templates/public-scaffold/` to `<parent>/<name>-public/`.

### 5b. Substitute placeholders

Same find-replace as 4c, on the public-side files. The placeholders in `templates/public-scaffold/` are a subset (mostly `{{PROJECT_NAME}}` and `{{PROJECT_DESCRIPTION}}`). Public side has no `CUSTOMIZE.md` or `blocked-paths.txt`, so no exclusions.

The vendored `templates/public-scaffold/CLAUDE.md` and `README.md` use `<fill in — e.g. X>` placeholders that the simple `{{}}` substitution doesn't touch. Don't worry about them — 5c and 5d overwrite both files entirely with content built from interview answers.

### 5c. Write public-side CLAUDE.md

Generate fresh from interview answers (do NOT try to patch the vendored placeholder version). It's a stripped variant of the private-side CLAUDE.md:

- Same project description (Q2)
- Same "What this is NOT" (Q4) — except entries the user flags as private-only (none flagged in v1; consider a Q4 follow-up "any of these private-side-only?" in a future revision)
- Same tech stack (Q9, Q10, Q12)
- Same conventions (Q15, Q16)
- Same gotchas (Q21) — except any flagged private-only
- Same external references (Q23) — strip URLs to local-disk paths (`~/notes/...`)
- AI collaboration rules: include only the universal ones (`no git add -A`, `no skip hooks`, `no --dangerously-skip-permissions`); strip pipeline-specific rules (no `improve.md`, no architect to consult)
- **Strip entirely:** "Data sensitivity" section (public side ships clean by definition)
- **Strip entirely:** "Pipeline" section (public side has no pipeline)

Write to `<parent>/<name>-public/CLAUDE.md`, overwriting the placeholder version copied from `templates/public-scaffold/`.

### 5d. Write public-side README.md

Generate fresh from interview answers. Do NOT substitute the public-scaffold's `<fill in>` placeholders — overwrite the file.

Structure: same as 4i but the "What's here" section reflects the static-scope public layout (worker.md, core-workflow, run.md), not the pipeline. No "Improvement pipeline" section. License default = `MIT` if Q3 ∈ {oss, customers}, else carry over private-side license choice.

### 5e. git init (public side)

`git init -b main` in `<parent>/<name>-public/`. No remote.

---

## Phase 6 — Report

Return a structured summary:

```
Skill: new-project
Status: SUCCESS | ABORTED | FAILED

On SUCCESS:
- Project name: <name>
- Layout: <single | dual>
- Pipeline included: <yes | no>
- Created paths:
  - <parent>/<name>-private/      (private fork, X files)
  - <parent>/<name>-public/       (public fork, Y files)   [dual only]
  OR
  - <parent>/<name>/              (single repo, X files)
- Interview questions asked: N
- Sensitive paths added to: blocked-paths.txt | .gitignore | (none — Q5 was none)
- CUSTOMIZE.md removed: yes | yes (already pruned with pipeline=no)
- Suggested next steps:
  1. cd <primary-dest>
  2. Open CLAUDE.md and confirm scope/AI rules look right.
  3. Open .claude/agents/architect.md and verify the scope-rules section reflects your real intent.   [pipeline only]
  4. Copy .claude/settings.example.json → .claude/settings.json and adjust permissions.
  5. Set a license in README.md (currently `<fill in>`) before any public push.   [if Q3 ∉ {oss, customers} so license placeholder remains]
  6. Add your project's source. When ready to publish, add a remote and substitute {{PUBLIC_REPO_URL}} in publisher.md.   [dual only]

On ABORTED:
- Phase: <Preconditions | Layout | Interview | Preview | Execute>
- Reason: <user said no | collision | interview aborted | etc.>
- Disk state: <no changes | partial — <path> exists but is incomplete>

On FAILED:
- Phase: <which>
- Error: <exact issue>
- Disk state: <what got written before the failure; manual cleanup may be needed>
- Recommended next step: <e.g. "rm -rf <partial-path> and rerun">
```

---

## Project-specific landmines

- **Don't `git add` or `git commit`.** This skill `git init`s only. The user decides what to stage and when to commit. Skipping commits prevents accidentally committing a half-bootstrapped state.
- **Don't add a remote.** The user adds remotes when they're ready to publish — keeping it absent prevents accidental early pushes of placeholder content.
- **Refuse if target exists and is non-empty.** It's much cheaper to ask the user to rename than to figure out what got partially overwritten.
- **The vendored placeholder scope rules in `architect.md` are intentional.** Replace them with interview-derived rules during 4d. Don't leave them in.
- **For single-layout projects, don't substitute `{{PUBLIC_REPO_ROOT}}` to a non-existent public path.** Either substitute to the same path as private (treating the single repo as both), or delete references entirely. The vendored `CUSTOMIZE.md` §2 explains which sections to delete for single-fork — follow that.
- **Windows path handling.** Use forward slashes inside generated markdown files (`CLAUDE.md`, agent profiles) for portability. Use `path.normalize` (or whatever the equivalent is in shell) for actual file system operations.
- **Public-side files should never reference `<parent>/<name>-private/`.** That path doesn't exist for forkers. Strip private references during 5b/5c.
- **Don't substitute `{{AI_COAUTHOR_IDENTITY}}` if the user picked `none`.** Delete the entire co-author block per `CUSTOMIZE.md` §1, including the trailing blank line.
