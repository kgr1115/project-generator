---
name: interview-project-intent
description: Conduct the adaptive project-intent interview. Called by /new-project (full bootstrap) and /generate-claudemd (retrofit). Asks one question at a time from the question bank, branches on answers, returns a structured answer dict. Not user-invocable directly — it's a sub-skill the orchestrating skills depend on.
argument-hint: [interview-mode = "greenfield" | "retrofit"]
---

Input: `$ARGUMENTS` is the interview mode (`greenfield` or `retrofit`). Mode affects which questions fire (Tier 3 layout/pipeline questions only fire for `greenfield`) and the cap.

---

## When to use this skill

Called by `new-project` and `generate-claudemd` skills. Not user-invocable directly. The user is interacting with the orchestrating skill; this skill is the engine that runs the Q&A.

The question catalog is in `templates/question-bank.md`. **That file is the source of truth for wording, defaults, and branching rules.** This skill's job is to execute the catalog correctly — not to redefine it.

---

## Phase 0 — Initialize

1. **Read the mode** from `$ARGUMENTS`. If empty, default to `greenfield` and warn.
2. **Set the cap:** `greenfield` = 15 questions max. `retrofit` = 10.
3. **Initialize an answer dictionary** keyed by question ID (Q1–Q23). Empty.
4. **Receive any pre-scan inferences** from the calling skill (passed via prior context). For each inferred answer, store it as `pre-filled` with the source manifest line — these will surface for confirmation rather than re-asking.

---

## Phase 1 — Run the interview

Execute the question bank in tier order: Tier 1 → 9. For each question:

1. **Check the trigger** (from the catalog). Skip if it doesn't fire.
2. **Check pre-fills.** If the calling skill pre-filled an answer for this Q (e.g. retrofit detected `pnpm` from `pnpm-lock.yaml`), surface it for confirmation rather than asking from scratch:
   - Confirmation prompt: "I see you're using pnpm based on `pnpm-lock.yaml` — confirm? [y / edit]". Single confirmation, no `AskUserQuestion` needed.
   - On `edit`, fall through to a normal ask.
3. **Ask the question** per the catalog's `Mode`:
   - **`AskUserQuestion`:** use the tool with the catalog's defaults. Always include an "other" or "skip" option where the catalog specifies one.
   - **`free-text`:** plain prompt; user replies in chat.
4. **For non-obvious answers, follow up with one short "why":** "Any reason — past bug, team preference, tooling constraint? Helps me write a more durable rule." Skip the why if the answer is conventional (e.g. "use 2-space indent" doesn't need a why; "use 4-space indent for JS" does).
5. **Store the answer.** Capture both the value and the optional rationale.
6. **Increment the question counter.** If it would exceed the cap, drop remaining `only-if-relevant` questions and proceed to Phase 2.
7. **If the user says "stop" / "skip everything" / "I'm done"** at any point: stop. Proceed to Phase 2 with whatever's captured.

### Branching rules (executed inline, per the catalog)

- **Q3 = `solo`** → skip Q17, Q18 (workflow tier)
- **Q5 = `none`** → skip Q5a (blocked paths seed)
- **Q9 = `python`** → skip Q11 (package manager); use Python-flavored options for Q10, Q12
- **Q9 ∉ {typescript, javascript}** → skip Q11
- **mode = `retrofit`** → skip Q6, Q7, Q8 (layout/pipeline are bootstrap-only)
- **Q9 inferred from manifest** → don't ask Q9 fresh; surface for confirmation

### Hard rules during the interview

- **One question at a time.** Never batch.
- **Use the catalog's wording verbatim** (or near-verbatim). Stable wording = predictable behavior.
- **Don't volunteer your own questions** outside the catalog. If you find yourself wanting to ask something not in the bank, stop the interview, propose adding it to `templates/question-bank.md` first, and ask the user if it's worth adding.
- **Don't argue with answers.** "Solo project, never published" → believe it; skip workflow tier.
- **Cap depth.** Stop at 15 (greenfield) or 10 (retrofit). The user's project doesn't deserve a 30-question gauntlet.

---

## Phase 2 — Validate & summarize

Before returning to the calling skill, do a sanity pass on the captured answers:

1. **Required answers present?** Q1, Q2, Q3, Q4 must all have non-empty values. If any are missing (rare — only happens if user aborted very early), return `status: incomplete` with the list of missing IDs to the calling skill so it can decide whether to retry or abort.
2. **Conflicts?** Check for obvious contradictions:
   - Q3 = `solo` but Q20 includes `no_push_without_approval` for a public OSS workflow → not a real conflict, but worth flagging.
   - Q5 = `customer_pii` and Q6 = `single` (single-repo) → flag: "You're handling PII but using a single-repo layout. Confirm you understand the public-side absence means you're responsible for not pushing this repo publicly."
   - Q7 = `no` (no pipeline) and answers in Tier 7 (AI collab) reference pipeline behaviors → fine, but trim the irrelevant references during assembly.
   - Surface conflicts to the user once, accept their reply, and move on.
3. **Build the structured answer object** — see Phase 3 for shape.

---

## Phase 3 — Return

Return a structured answer dict to the calling skill. Format:

```yaml
status: complete | partial | aborted
mode: greenfield | retrofit
question_count: <N>
cap: <N>
answers:
  Q1_project_name: "<value>"
  Q2_one_line_description: "<value>"
  Q3_end_user: "<enum value>"
  Q3_end_user_other: "<free-text follow-up if applicable>"
  Q4_scope_nots:
    - "<bullet 1>"
    - "<bullet 2>"
  Q5_data_sensitivity: "<enum value>"
  Q5a_blocked_paths_seed:    # only if Q5 ≠ none
    - "<path or glob>"
  # ... etc, one entry per asked question
rationales:
  Q15_style_deviations: "<the 'why' the user gave, if any>"
  # ... only entries that have a recorded why
inferred:
  Q9_primary_language: { value: "typescript", source: "package.json" }
  Q11_package_manager: { value: "pnpm", source: "pnpm-lock.yaml" }
  # ... only entries that came from pre-scan and were confirmed
skipped:
  - Q17_branch_strategy: "Q3=solo"
  - Q22_domain_glossary: "user said 'skip'"
  # ... reason per skipped question
conflicts_flagged:
  - "Q5=customer_pii but Q6=single — user confirmed they understand"
```

The calling skill is responsible for using this to fill templates, write files, etc. This skill never writes files itself.

---

## Phase 4 — Report

After returning, surface a short summary to the user:

```
Interview complete.
- Mode: <mode>
- Questions asked: <N> (out of cap <C>)
- Required answers: ✓ all captured
- Skipped: <count> (not applicable based on prior answers)
- Inferred from manifests: <count> (confirmed)
```

Then yield control back to the calling skill, which proceeds to template assembly and disk writes.

---

## Project-specific landmines

- **`AskUserQuestion` returns one selection per call.** For Q20 (multi-select forbidden actions), either ask sequentially per option ("Apply rule X? [y/n]") OR present the list as free-text "comma-separated". Don't try to express multi-select in a single `AskUserQuestion`.
- **Free-text follow-ups need explicit framing.** When asking "any reason?", the user often replies with the actual answer instead of the rationale. Read replies forgivingly — if the reply contains the rationale, store it; if it contains an updated answer, treat it as a correction.
- **Pre-scan inferences must be confirmed, not assumed.** A `package.json` with TypeScript files might still be a Bun project, an Express project, or a JSX-without-TS project. Always surface for `[y / edit]`, never silently fill.
- **Don't read source code in the interview.** Read manifests only (`package.json`, `pyproject.toml`, etc.). The pre-scan happens in the calling skill before this one is invoked, so all reads should already have happened.
- **The "why" follow-up is asked at most once per question.** If the user says "no reason" or gives a non-answer, accept it and move on — don't re-ask.
- **The cap is a ceiling, not a target.** Don't pad questions to reach the cap. A 6-question retrofit interview is fine if the manifests covered most of the spine.
