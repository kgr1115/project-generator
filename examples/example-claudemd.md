<!--
EXAMPLE OUTPUT — what /new-project or /generate-claudemd produces for a fictional project.

Project: hn-digest
Use case: A personal CLI that pulls the top 10 Hacker News stories every morning
and emails them to the user. Single repo, pipeline opted in, real data on-disk
(emails, HN API key), no public fork. Roughly 130 lines after assembly.

Use this file as a reference for what good generator output looks like. Regenerate
after structural changes to templates/claudemd-template.md.
-->

# hn-digest — standing brief

> Read this first when working in this repository.

## What this project is

A personal CLI that pulls the top 10 Hacker News stories every morning and emails them to me with a one-line LLM-generated summary per story.

Single user (me). Runs on my laptop via cron. Local-only — no hosting, no other users.

## What this project is NOT

- **Not a SaaS or multi-user product.** No accounts, no auth, no telemetry.
- **Not a paid service.** Only budgeted cost is LLM tokens — capped at $5/month. No paid HN APIs (use the free Algolia HN API), no hosted infra, no managed databases.
- **Not a public artifact.** This repo holds my email and HN API key in `.env`. Never push to a public remote.

## How to run it

```
pip install -r requirements.txt
python -m hn_digest                  # one-shot run
python -m hn_digest --dry-run        # build the digest, don't send
pytest                                # tests
```

## Tech stack

- **Language:** Python 3.13
- **Framework:** stdlib + `httpx` for the HN API, `jinja2` for the email template
- **Test runner:** `pytest`
- **LLM:** Anthropic API (Haiku), via `anthropic` SDK

## Conventions

- Use `pathlib.Path`, never string `+` for paths.
- Keep all I/O behind functions in `hn_digest/io.py` so tests can swap implementations without mocking the network.
- One file per concern; no `utils.py` catch-all.

## AI collaboration rules

- **IMPORTANT:** Never push without my explicit go-ahead. This repo holds my email address and HN API key.
- **IMPORTANT:** Never `git add -A` or `git add .`. Stage files explicitly.
- **IMPORTANT:** Never modify files in `data/sent/` — those are the historical record of what got emailed.
- Never call paid APIs (anything beyond Anthropic Haiku) without asking.
- Never skip commit hooks (`--no-verify`, `--no-gpg-sign`).
- For changes touching `hn_digest/email.py`, use plan mode first — silent send-failures cost me trust in the digest.

## Gotchas / known landmines

- **The HN Algolia API rate-limits at 100 requests/minute.** A naive loop fetching 10 story details = 11 requests, fine, but anything that fans out beyond that needs throttling. See `hn_digest/api.py:fetch_top` for the canonical pattern.
- **Timestamps in `data/sent/*.jsonl` are UTC.** Everywhere else (logs, terminal output) uses local time. Don't mix them — it's bitten me twice when debugging "why didn't the digest send."
- **My SMTP provider (Fastmail) bounces messages over 100KB.** The digest template truncates each summary to 200 chars; don't loosen this without testing the full send path.
- **`anthropic` SDK retries are off by default.** If you set `max_retries`, also set a timeout — I've had it hang for 12 minutes on a transient network error.
- **`os.replace()` only on Windows for moving the digest log.** `shutil.move()` fails silently across drives — I store digests on `D:` and run on `C:`.

## Data sensitivity

This repo holds:
- My email address (in `.env` — gitignored)
- My HN account credentials (also in `.env`)
- The send-history log at `data/sent/*.jsonl` (real recipient = me, but still don't push)

The `blocked-paths.txt` enforces that `.env` and `data/sent/` never get staged. Don't disable the publisher's data guard.

## Glossary

- **Digest** — the assembled email body for a given day, before sending.
- **Story object** — the dict returned by `hn_digest.api.fetch_story(id)`. Has `id`, `title`, `url`, `author`, `score`, `summary`.
- **Send log** — `data/sent/<YYYY-MM-DD>.jsonl`. One line per story sent that day. Used to dedupe across runs.

## References

- HN Algolia API docs: https://hn.algolia.com/api
- Anthropic API: see `@requirements.txt` for pinned version
- Fastmail SMTP setup notes: `~/notes/fastmail-app-passwords.md` (local only)

## See also

- `@README.md` — short user-facing README (explains how to set up `.env`, run cron entry)
- `@ARCHITECTURE.md` — pipeline philosophy + handoff contracts (vendored from project-generator)
- `@CUSTOMIZE.md` — downstream customization checklist (largely already applied during bootstrap)
