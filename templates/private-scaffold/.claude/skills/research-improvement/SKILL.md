---
name: research-improvement
description: Audits the {{PROJECT_NAME}} codebase end-to-end and researches external precedent to produce up to 10 prioritized, concrete improvement proposals. Use when an improvement cycle is starting (e.g. `/improve`, "run the improvement pipeline", "audit and improve") and the architect needs a structured proposal set to gate.
argument-hint: [focus area — e.g. "UI" or "onboarding friction" — or omit for full audit]
---

Focus area (if any): `$ARGUMENTS`

---

## Phase 1 — Audit (read-only pass through the live codebase)

Trace the project's core user flow in order. Identify the flow from {{PROJECT_NAME}}'s standing brief or README if it's not obvious.

For each step, ask: **is there friction the user would pay complexity to remove?**

### Where to look (adapt to this project's actual surfaces)

| Surface | Files to read |
|---|---|
| Primary UI (if any) | Rendered output + generator/template |
| Background automation | Watchers, cron, schedulers, pipelines |
| Schema / data model | Shared data files, column headers, schema docs |
| Slash commands | `.claude/commands/*.md` — descriptions clear enough for routing? |
| Agent profiles | `.claude/agents/*.md` — steps actionable? outputs well-defined? |
| Onboarding | README, setup docs — can a fresh clone reach "ready" quickly? |
| Docs vs. behavior | Compare standing-brief workflow descriptions to actual code |

### Friction checklist

- Clicks / copy-paste that could be automated
- Missing visual state cues (stale items, error states, loading indicators)
- Unclear status transitions in shared data
- Agent profiles with vague or overlapping descriptions
- Scripts that fail silently or produce unhelpful errors
- Onboarding steps that require out-of-band knowledge

---

## Phase 2 — External research

Use WebSearch/WebFetch to study adjacent tools and UX conventions. Filter everything against the hard constraints before including it.

Good search angles (adapt to the project):
- How do adjacent/competitor tools in this space structure their equivalent of the core flow?
- What UX conventions exist for this kind of interface?
- What library/architectural patterns make this kind of pipeline less brittle?
- What recent techniques reduce failure rates in this domain?

**Hard constraints — never recommend anything that:**
- Requires a paid service or dependency the project disallows (check the architect's scope doc).
- Requires install channels the project disallows.
- Adds telemetry, accounts, auth, or multi-user concepts (if the project is single-user).
- Needs scope > ~5 files in one proposal.
- Touches read-only source material.

---

## Phase 3 — Synthesize into proposals

Produce up to 10 proposals. One finding → one proposal. Be specific: name the file, the behavior change, and the user-facing impact. "Improve UX" is not a proposal.

### Output format (submit to architect)

```markdown
# Research — {YYYY-MM-DD}

## Proposal N: {short title}
**Category:** reliability | UX | core-flow | data | onboarding | docs | other
**Why it matters for the user:** {one paragraph — user-facing impact}
**Concrete change:** {1-2 paragraphs — which files, what behavior change, no hand-waving}
**Sources / precedent:** {cite if drawn from external research}
**Estimated scope:** single file | multi-file | schema | dependency
**Risk:** {what could break}
**Priority:** P0 reliability blocker | P1 high UX value | P2 polish | P3 nice-to-have
```

---

## Common failure modes for this role

- **Proposing scope creep** — full rewrites, new hosted services, unrelated features. If you catch yourself proposing something that isn't tighter/clearer/more reliable for the core flow, cut it.
- **Vague proposals** — the architect will return vague proposals for revision, costing a round trip. Name the file; name the behavior.
- **Recommending forbidden dependencies** — check the architect's scope doc before proposing anything that adds a dependency.
- **Touching user data during research** — read-only always.
- **Missing the docs-vs-behavior gap** — the standing brief often describes workflows that diverged from the actual code. These gaps are high-value P1 proposals.

---

## When you find nothing high-leverage

Valid output: "Found no high-leverage improvements this cycle. Highest-priority findings are P3/nice-to-have. Recommend pausing for N cycles or revisiting when user behavior reveals new pain points."
