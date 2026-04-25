---
description: Generate a tailored CLAUDE.md for a project by interviewing the user one question at a time. Detects greenfield vs. retrofit, branches on answers, previews before writing. Triggered by /generate-claudemd or natural-language phrases like "generate a CLAUDE.md", "set up Claude memory for this repo", "make a CLAUDE.md for this project".
argument-hint: [optional target directory — e.g. "../my-new-project" — omit to target the current working directory]
---

The user wants a tailored CLAUDE.md for a project. Invoke the `generate-claudemd` skill with the argument `$ARGUMENTS`.

The skill will:
1. Resolve the target directory (`$ARGUMENTS`, or cwd if omitted) and detect greenfield vs. retrofit mode.
2. Pre-scan manifests (`package.json`, `pyproject.toml`, etc.) for retrofits to skip already-answered questions.
3. Run an adaptive one-question-at-a-time interview covering identity, scope, stack, conventions, workflow, AI collaboration preferences, and gotchas.
4. Assemble a tight (<200 line) CLAUDE.md from `templates/claudemd-template.md`, stripping empty sections.
5. Preview the full output in chat, accept revisions, then write to `<target>/CLAUDE.md` (with `.bak` backup if one already exists).

Follow the skill exactly — do not improvise the question order or skip the preview phase.
