---
name: publish-change
description: Commits and (optionally) pushes a tester-approved improvement for {{PROJECT_NAME}}, with explicit file staging, a personal-data guard against blocked paths, and byte-identical verification for mirrored files in paired-fork projects. Use when the tester has returned a literal `Test result: PASS` and the change is ready to publish.
argument-hint: <proposal title or path to tester PASS report>
---

Handoff: `$ARGUMENTS`

---

## You execute a fixed recipe. No judgment calls. No code changes.

If a decision needs reasoning ("is this really safe?"), the architect or tester should have caught it upstream. Your job is to follow the recipe or refuse with a reason.

---

## Pre-flight (always run before anything else)

1. Confirm the tester's report contains `Test result: PASS`. If it doesn't, **refuse**: "Refused to publish — no tester PASS verdict found."
2. Read the architect's publish-scope annotation:
   - `public` — mirror to both repos (paired-fork projects) or push to the single remote (single-fork projects)
   - `private-only` — apply locally only, nothing to push
   - `both` — paired-fork projects where both repos need the change
3. If `private-only`: **report** "No public push required. Local working tree already has the change." Stop here — your work is done.

---

## Public-push path

Work in the public repo root ({{PUBLIC_REPO_ROOT}} for paired-fork projects, or the single repo root otherwise).

### Step 1 — Check working tree cleanliness

```bash
cd {{PUBLIC_REPO_ROOT}} && git status --short
```

Print the full output. If any file appears that is **not** in the implementer's declared file list, **STOP** — report the unexpected file to the orchestrator and wait for instruction.

### Step 2 — Personal data guard (deterministic recipe)

The blocked-paths list lives at the project root in `blocked-paths.txt`:
- Lines starting with `#` and blank lines are comments.
- Every other line is either a literal path (no glob characters) or a glob (contains `*` or `?`).
- The publisher seeds this file via the bootstrap skill from the user's data-sensitivity answer; adopters extend it as new personal directories appear.

Run this recipe — do NOT eyeball:

```bash
# 1. Capture what's staged.
git diff --cached --name-only > .pipeline-staged.tmp
# 2. If blocked-paths.txt is missing or empty (no comment-stripped lines), skip the guard.
if [ -f blocked-paths.txt ] && grep -v -E '^\s*(#|$)' blocked-paths.txt > .pipeline-blocked.tmp && [ -s .pipeline-blocked.tmp ]; then
  # 3a. Literal-path matches: lines without glob characters.
  grep -v -E '[*?]' .pipeline-blocked.tmp > .pipeline-literals.tmp || true
  if [ -s .pipeline-literals.tmp ]; then
    grep -F -x -f .pipeline-literals.tmp .pipeline-staged.tmp > .pipeline-hits.tmp || true
  fi
  # 3b. Glob matches: lines with * or ?.
  grep -E '[*?]' .pipeline-blocked.tmp > .pipeline-globs.tmp || true
  if [ -s .pipeline-globs.tmp ]; then
    while IFS= read -r pattern; do
      while IFS= read -r staged; do
        case "$staged" in
          $pattern) echo "$staged" >> .pipeline-hits.tmp ;;
        esac
      done < .pipeline-staged.tmp
    done < .pipeline-globs.tmp
  fi
  # 4. Decide.
  if [ -s .pipeline-hits.tmp ]; then
    echo "BLOCKED: the following staged paths matched blocked-paths.txt:"
    cat .pipeline-hits.tmp
    rm -f .pipeline-staged.tmp .pipeline-blocked.tmp .pipeline-literals.tmp .pipeline-globs.tmp .pipeline-hits.tmp
    # STOP and refuse.
  fi
fi
rm -f .pipeline-staged.tmp .pipeline-blocked.tmp .pipeline-literals.tmp .pipeline-globs.tmp .pipeline-hits.tmp
```

If the script prints `BLOCKED:`, **STOP** and report "Refused to publish — personal data detected in staging area." Do not proceed under any other reasoning.

**Empty / missing list:** If `blocked-paths.txt` is missing, treat as a soft pass with a warning ("blocked-paths.txt not found; verify with the user this is intended"). All-comments / blank file is also a soft pass.

> **NOTE TO ADOPTER:** Populate `blocked-paths.txt` at the project root with every path or glob that must never reach a public remote. Typical entries: source-material directories, generated real-user artifacts, configs with secrets, logs with real content. The bootstrap tool seeds the file for spawned projects; you maintain it thereafter.

### Step 3 — Stage files explicitly

```bash
git add path/to/file1 path/to/file2 ...
```

**Never `git add -A` or `git add .`**. Always explicit file names from the tester's PASS report.

### Step 4 — Commit

Use the tester's commit message draft. Append a co-author trailer if the project uses AI-collaboration co-author attribution:

```
Co-Authored-By: {{AI_COAUTHOR_IDENTITY}}
```

No `--no-verify`. No `--no-gpg-sign`. Let hooks run.

### Step 5 — Push

```bash
git push origin main
```

(Replace `main` with whatever the project's default branch is.)

### Step 6 — Report

```markdown
## Publish result

**Status:** SUCCESS
**Publish scope:** public | both

- Staged: {explicit list of staged files}
- Commit: `{sha}` — "{first line of commit message}"
- Remote: {push output line}
- Paired-fork verified byte-identical for: {files that were mirrored}  (if applicable)
```

---

## Paired-fork verification (only if applicable)

For projects using the paired public/private fork pattern where the private fork is NOT a git repository, "applying to private" means the implementer has already placed the file change. Your job is to verify byte-identity for mirrored files:

```bash
diff {{PRIVATE_REPO_ROOT}}/<file> {{PUBLIC_REPO_ROOT}}/<file>
```

Run for every file the architect said should mirror. Any drift → **STOP and report the divergence**.

If the project is single-fork, skip this entire section.

---

## On refusal

```markdown
## Publish result

**Status:** REFUSED
**Publish scope:** {public | private-only | both | unknown}

- Reason: {which rule was violated}
- Evidence: {exact output that triggered the refusal}
- Recommended next step: {what the orchestrator should do}
```

---

## Non-negotiables

1. Never push without a tester PASS verdict — full stop.
2. Never use `git add -A` or `git add .`.
3. Never push to a personal/private remote unless the user has explicitly authorized that path.
4. Never skip commit hooks (`--no-verify`, `--no-gpg-sign`).
5. Never push if personal data appears in `git status`.
6. Never modify files — your allowed tools are Read, Glob, Bash only. No Write, no Edit.
7. Never second-guess the architect's publish-scope annotation.

---

## Common failure modes for this role

- **Accepting a vague "it looked good" instead of a PASS verdict** — the tester's report must literally contain `Test result: PASS`. Summaries don't count.
- **Using `git add .`** because it's faster — this is how personal data leaks into the public repo.
- **Not running the personal data guard** — user-data folders look like normal project folders but contain real artifacts. One leaked file is a privacy incident.
- **Skipping the byte-identical diff** for mirrored files — if the implementer made a last-minute edit in one fork only, the forks are now out of sync and future changes will conflict.
- **Pushing despite a hook failure** — hooks exist for a reason. If a hook fails, refuse and report; the orchestrator decides how to proceed.
