# Question Bank

The canonical catalog of questions the interview skill (`interview-project-intent`) draws from. Each entry has:

- **ID** — stable identifier the skill references.
- **Tier** — coarse grouping; tiers are answered roughly in order.
- **Trigger** — when the question fires (`always`, or a condition based on prior answers).
- **Mode** — `AskUserQuestion` (multiple choice) or `free-text`.
- **Wording** — what the LLM literally asks. Stable wording = stable behavior.
- **Defaults** — for `AskUserQuestion`, the option list. Always include "other" / "skip" where reasonable.
- **Maps to** — which output files/sections the answer flows into.
- **Why** — the rationale (kept short; for maintainers, never read aloud).

Total spine: 20 questions. The skill caps the actual count at 12–15 for greenfield, 6–10 for retrofit, by skipping `Trigger: only-if-relevant` questions and pre-filling from manifests.

---

## Tier 1 — Identity (always asked)

### Q1 — project_name
- **Tier:** 1
- **Trigger:** always (skip if supplied as `$ARGUMENTS` to the slash command)
- **Mode:** free-text
- **Wording:** "What's the project's name? (used for the directory and as `{{PROJECT_NAME}}` in templates)"
- **Maps to:** `CLAUDE.md` title, `README.md`, every `{{PROJECT_NAME}}` placeholder, directory name
- **Why:** The single most-substituted token. Asked first because it's cheap and grounds everything else.

### Q2 — one_line_description
- **Tier:** 1
- **Trigger:** always
- **Mode:** free-text
- **Wording:** "Describe the project in one sentence — what it does, not what it's built with."
- **Maps to:** `CLAUDE.md` "What this project is", `README.md` opener
- **Why:** Forces the user to articulate intent before stack. "What it does, not what it's built with" prevents tech-stack answers.

### Q3 — end_user
- **Tier:** 1
- **Trigger:** always
- **Mode:** AskUserQuestion
- **Wording:** "Who's it for?"
- **Defaults:**
  - `solo` — Just me / personal use
  - `small_team` — Small team / shared internal tool
  - `oss` — Public open-source
  - `customers` — Paying customers / production product
  - `other` — Other (free-text follow-up)
- **Maps to:** `CLAUDE.md` "Who this is for", `orchestrator.md` `{{END_USER_DESCRIPTION}}`, `researcher.md` `{{END_USER_DESCRIPTION}}`
- **Why:** Drives many downstream decisions — `solo` skips workflow questions; `customers` increases scrutiny on data sensitivity.

---

## Tier 2 — Scope walls (always asked — most load-bearing tier)

### Q4 — scope_nots
- **Tier:** 2
- **Trigger:** always
- **Mode:** free-text
- **Wording:** "What's this project NOT? Two or three things it should never become — cost ceilings, scope walls, infrastructure you won't adopt. (This is the most important answer in the interview — it becomes the architect's scope rules and the 'What this project is NOT' section of CLAUDE.md.)"
- **Maps to:** `CLAUDE.md` "What this project is NOT", `architect.md` scope rules, `architect-review/SKILL.md` scope rules
- **Why:** Without explicit scope walls, an LLM defaults to "yes, let's add that." Eliciting "no"s up front prevents months of scope creep.

### Q5 — data_sensitivity
- **Tier:** 2
- **Trigger:** always
- **Mode:** AskUserQuestion
- **Wording:** "What kind of data does the project handle?"
- **Defaults:**
  - `none` — No real data, only synthetic/test
  - `personal_local` — Personal data on disk, never pushed (notes, contacts, local files)
  - `customer_pii` — Real customer/PII data (emails, names, payment info)
  - `regulated` — Regulated data (HIPAA, PCI, GDPR-restricted)
  - `secrets_only` — No user data, but holds API keys/credentials
- **Maps to:** `CLAUDE.md` "Data sensitivity" section, `blocked-paths.txt` (seeds typical patterns based on category), `publisher.md` data-guard emphasis
- **Why:** Same answer drives `blocked-paths.txt` content, dual-vs-single layout recommendation, and CLAUDE.md sensitivity section. Strong signal that affects multiple files.

### Q5a — blocked_paths_seed
- **Tier:** 2
- **Trigger:** Q5 ≠ `none`
- **Mode:** free-text (multiline)
- **Wording:** "List specific paths or globs that should never be pushed (one per line). E.g. `data/originals/*`, `config/credentials.json`, `logs/*.md`. Skip if you'll define them later."
- **Maps to:** `blocked-paths.txt`
- **Why:** Lets the user pre-populate the personal-data guard. Empty answer is fine — they can fill it in later.

---

## Tier 3 — Layout & pipeline (asked only by /new-project)

### Q6 — layout
- **Tier:** 3
- **Trigger:** command = `/new-project` (skipped by `/generate-claudemd`)
- **Mode:** AskUserQuestion (no default — user must choose)
- **Wording:** "Repo layout? `dual` creates paired `<name>-private/` (working copy with real data) and `<name>-public/` (clean, shippable). `single` creates one repo. Pick `single` if the project never publishes or has no real-data concern."
- **Defaults:**
  - `dual` — Paired private + public
  - `single` — One repo
- **Maps to:** Determines whether `templates/public-scaffold/` is copied; affects which placeholders are substituted; affects CLAUDE.md "Data sensitivity" section content
- **Why:** Cannot be inferred. Must be an active choice — defaulting silently to either is wrong for half of users.

### Q7 — pipeline_optin
- **Tier:** 3
- **Trigger:** command = `/new-project` (skipped by `/generate-claudemd`)
- **Mode:** AskUserQuestion
- **Wording:** "Include the agent improvement pipeline (researcher → architect → implementer → tester → publisher → debugger)? It's powerful but adds 8 agent files + 6 skills to the private side. Skip for one-shot scripts or trivial repos."
- **Defaults:**
  - `yes` — Include the full pipeline
  - `no` — Skip; just `.claude/settings.example.json` and `CLAUDE.md`
- **Maps to:** If `no`: prune `.claude/agents/`, `.claude/skills/`, `.claude/commands/improve.md`, `ARCHITECTURE.md`, `CUSTOMIZE.md`, `blocked-paths.txt` from the private side after copy
- **Why:** Most projects want the pipeline. Some don't (e.g., a config repo, a one-shot script). Asking respects that and keeps small projects small.

### Q8 — coauthor_attribution
- **Tier:** 3
- **Trigger:** command = `/new-project`, Q7 = `yes`
- **Mode:** AskUserQuestion
- **Wording:** "AI co-author attribution for commits the publisher makes?"
- **Defaults:**
  - `claude` — `Claude <noreply@anthropic.com>`
  - `none` — No attribution line
  - `custom` — Custom (free-text follow-up)
- **Maps to:** `publisher.md`, `publish-change/SKILL.md` `{{AI_COAUTHOR_IDENTITY}}` placeholder
- **Why:** Trivially small but the substitution is annoying to do manually post-bootstrap.

---

## Tier 4 — Tech stack

### Q9 — primary_language
- **Tier:** 4
- **Trigger:** always (auto-detected for retrofit if a manifest exists; surfaced for confirmation)
- **Mode:** AskUserQuestion (or free-text confirmation if pre-detected)
- **Wording:** "Primary language?" (or "I see this is a TypeScript project — confirm? [y/edit]")
- **Defaults:**
  - `typescript`, `javascript`, `python`, `go`, `rust`, `java`, `csharp`, `ruby`, `other`
- **Maps to:** `CLAUDE.md` tech-stack section; gates which follow-up questions ask
- **Why:** Single question that prunes the entire question tree.

### Q10 — runtime_framework
- **Tier:** 4
- **Trigger:** always (branched by Q9)
- **Mode:** AskUserQuestion (defaults vary by language)
- **Wording:** depends on Q9 — e.g.:
  - TS/JS: "Runtime + framework?" → Node (Express/Fastify/Next.js/none), Bun, Deno
  - Python: "Runtime + framework?" → 3.13/3.14 + Django/FastAPI/Flask/none
  - Go: "Go version + framework?" → 1.22/1.23 + chi/gin/none
- **Maps to:** `CLAUDE.md` tech-stack section
- **Why:** Together with Q9 establishes enough context to skip language-specific questions later.

### Q11 — package_manager
- **Tier:** 4
- **Trigger:** Q9 ∈ {typescript, javascript} (auto-detected via lockfile if retrofit)
- **Mode:** AskUserQuestion
- **Defaults:** `pnpm`, `npm`, `yarn`, `bun`
- **Maps to:** `CLAUDE.md` tech-stack section, run/test commands
- **Why:** Needed to write correct command examples in CLAUDE.md.

### Q12 — test_runner
- **Tier:** 4
- **Trigger:** always (auto-detected if retrofit)
- **Mode:** AskUserQuestion (branched by Q9)
- **Defaults vary:** e.g. `vitest`, `jest`, `node:test` for TS/JS; `pytest`, `unittest` for Python
- **Maps to:** `CLAUDE.md` tech-stack + "How to run it"
- **Why:** Tests are how Claude verifies its work — this is the single most important command in CLAUDE.md after `run`.

### Q13 — run_command
- **Tier:** 4
- **Trigger:** always
- **Mode:** free-text
- **Wording:** "What's the exact command to run the project locally? (e.g. `npm run dev`, `python main.py`, `cargo run --bin server`)"
- **Maps to:** `CLAUDE.md` "How to run it"
- **Why:** Specific verbatim commands are non-negotiable per Anthropic's guidance.

### Q14 — install_command
- **Tier:** 4
- **Trigger:** always (auto-derivable for retrofit; may skip)
- **Mode:** free-text
- **Wording:** "Install command? (e.g. `pnpm install`, `pip install -r requirements.txt`)"
- **Maps to:** `CLAUDE.md` "How to run it"
- **Why:** First command anyone runs in a fresh checkout.

---

## Tier 5 — Conventions (each is `only-if-relevant`)

### Q15 — style_deviations
- **Tier:** 5
- **Trigger:** always (with explicit "skip if standard" framing)
- **Mode:** free-text
- **Wording:** "Any code style that *deviates* from the language's defaults? (indentation width, naming, import style, anything Claude shouldn't assume.) Say 'standard' to skip."
- **Maps to:** `CLAUDE.md` "Conventions" section. If empty, the section is stripped entirely.
- **Why:** Per Anthropic's guidance, "exclude standard language conventions Claude already knows." Asking for *deviations only* keeps the section short.

### Q16 — comment_philosophy
- **Tier:** 5
- **Trigger:** only-if-relevant (skip for trivial projects)
- **Mode:** AskUserQuestion
- **Wording:** "Comment philosophy?"
- **Defaults:**
  - `minimal` — Only when the WHY is non-obvious
  - `normal` — Standard docstrings on public APIs
  - `heavy` — Explain everything, including obvious code
  - `skip` — No preference / don't add to CLAUDE.md
- **Maps to:** `CLAUDE.md` "Conventions" section (single bullet)
- **Why:** Comment style is a frequent friction point. One bullet in CLAUDE.md prevents repeated correction.

---

## Tier 6 — Workflow (skipped if Q3 = solo)

### Q17 — branch_strategy
- **Tier:** 6
- **Trigger:** Q3 ∈ {small_team, oss, customers}
- **Mode:** AskUserQuestion
- **Defaults:** `trunk` (commit to main), `feature_branch` (PR + squash-merge), `gitflow` (long-lived develop), `none`
- **Maps to:** `CLAUDE.md` "Workflow" section
- **Why:** Trivial to wrong. Wrong default ("just push to main") on a team project causes the kind of mistake that ruins trust.

### Q18 — commit_style
- **Tier:** 6
- **Trigger:** Q3 ∈ {small_team, oss, customers}
- **Mode:** AskUserQuestion
- **Defaults:** `conventional` (`feat:`/`fix:`/etc.), `prose` (plain English), `project` (free-text rule)
- **Maps to:** `CLAUDE.md` "Workflow" section
- **Why:** Same as Q17 — once-and-done.

---

## Tier 7 — AI collaboration (always asked, most undervalued tier)

### Q19 — autonomy_level
- **Tier:** 7
- **Trigger:** always
- **Mode:** AskUserQuestion
- **Wording:** "How autonomous should Claude be in this project?"
- **Defaults:**
  - `ask_always` — Ask before any non-trivial change
  - `ask_risky` — Ask only for risky/irreversible changes (default for most projects)
  - `auto` — Full auto mode; trust the safety rails
- **Maps to:** `CLAUDE.md` "AI collaboration rules" section
- **Why:** Sets the floor for how Claude operates. Different across projects (a paid-customer SaaS is `ask_always`; a personal scratchpad is `auto`).

### Q20 — forbidden_actions
- **Tier:** 7
- **Trigger:** always
- **Mode:** AskUserQuestion (multi-select via free-text follow-up)
- **Wording:** "Which of these forbidden actions apply? (Pick any combination — defaults shown.)"
- **Defaults (all checked by default for safety):**
  - `no_push_without_approval` — Never push without explicit go-ahead
  - `no_git_add_all` — Never `git add -A` or `git add .`
  - `no_skip_hooks` — Never `--no-verify` or `--no-gpg-sign`
  - `no_paid_apis_unilateral` — Never call paid APIs without approval
  - `no_dangerous_skip_perms` — Never `--dangerously-skip-permissions`
  - `no_data_mutation` — Never mutate user data without approval (additive for `personal_local`/`customer_pii` projects)
- **Maps to:** `CLAUDE.md` "AI collaboration rules" section. Reinforces vendored `publisher.md` safety rails.
- **Why:** Most teams have a story like "the LLM force-pushed and we lost two days of work." Capturing the explicit "never"s closes that gap.

---

## Tier 8 — Domain & gotchas (always asked, highest-value section)

### Q21 — gotchas
- **Tier:** 8
- **Trigger:** always
- **Mode:** free-text (multiline)
- **Wording:** "Any non-obvious gotchas — things that have already bitten you, surprising behaviors, fragile areas? List as many as you want, one per line. (This becomes the most-read section in CLAUDE.md. If you skip everything else, don't skip this.)"
- **Maps to:** `CLAUDE.md` "Gotchas" section, `debug/SKILL.md` Step 2 (failure-mode table), `implementer.md` Phase 4 (landmines)
- **Why:** The single highest-leverage answer. Each gotcha saves a future debugger round trip. Same answer feeds three files.

### Q22 — domain_glossary
- **Tier:** 8
- **Trigger:** only-if-relevant (skip for "no jargon" projects)
- **Mode:** free-text (multiline)
- **Wording:** "Any project-specific terms Claude should know? Internal product names, acronyms, jargon. Format: `term — definition`, one per line. Skip if standard."
- **Maps to:** `CLAUDE.md` "Glossary" section
- **Why:** Domain language is a frequent source of "Claude doesn't get what we mean." One short section fixes it.

---

## Tier 9 — References (asked once, at the end)

### Q23 — external_references
- **Tier:** 9
- **Trigger:** always (with "skip if none" framing)
- **Mode:** free-text (multiline)
- **Wording:** "External resources Claude should know about? Internal wiki URLs, monitoring dashboards, external API docs, issue trackers. One per line. Skip if none."
- **Maps to:** `CLAUDE.md` "References" section
- **Why:** Cheap to capture; saves repeated "where do I find X" exchanges.

---

## Branching summary

| Trigger | Effect |
|---|---|
| Q3 = `solo` | Skip Q17, Q18 |
| Q5 = `none` | Skip Q5a (blocked paths) |
| Q9 = `python` | Skip Q11 (package manager); ask Python-flavored Q10/Q12 |
| Q9 ≠ TS/JS | Skip Q11 |
| `/generate-claudemd` (retrofit) | Skip Q6, Q7, Q8 entirely |
| Pre-scan inferred answer | Surface for confirmation, don't re-ask |
| Project description suggests trivial scope | Mark Q16 as skippable; mark Q22 as skippable |

## Cap rules

- **Greenfield (`/new-project`):** hard cap 15 questions. If branching would exceed it, drop the lowest-tier `only-if-relevant` questions first.
- **Retrofit (`/generate-claudemd`):** hard cap 10 questions. Lean heavily on pre-scan.
- **The user's "stop" command always wins.** Generate from what's captured.

## Adding new questions

When adding:
1. Pick a tier (insert; don't disturb Q1–Q23 numbering — append as Q24, Q25, ...).
2. Specify ID, trigger, mode, wording, defaults, maps-to, why.
3. Update branching summary if the new question has a trigger.
4. Add a wiring step in `interview-project-intent/SKILL.md`.
5. Add the placeholder to `templates/claudemd-template.md` if the answer feeds CLAUDE.md.
6. Regenerate `examples/example-claudemd.md`.

When removing/changing:
1. Mark the entry deprecated rather than deleting (preserves the numbering).
2. Update the skill flow.
3. Note the deprecation in this bank's changelog at the bottom.

## Changelog

- **2026-04-25 — v1:** Initial 23-question bank.
