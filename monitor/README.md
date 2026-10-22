# Job monitor

Twice-daily scheduled scrape of EMEA junior-level SDE / MLE / Applied Scientist
roles at large tech companies. Net-new postings are pushed as a single digest
to your phone via [ntfy.sh](https://ntfy.sh).

Runs **locally only** (`python -m monitor.run`) and persists state in
`jobs.db`. The GitHub Actions cron (`.github/workflows/scrape.yml`) was removed
in Oct 2026 so the repo spends no Actions minutes; restore it from git history
if you want scheduled runs again.

## Setup

1. **Pick an ntfy topic.** Any string works — make it long and unguessable so
   nobody else can subscribe. Example: `jobs-feiyang-7xq2k9p`. There's no
   account; the topic *is* your authentication.
2. **Set the topic.** `export NTFY_TOPIC=your-topic-here` in the shell you
   run the monitor from.
3. **Subscribe on your phone.** Install the ntfy app
   ([iOS](https://apps.apple.com/us/app/ntfy/id1625396347) /
   [Android](https://play.google.com/store/apps/details?id=io.heckel.ntfy))
   and *Add subscription* → enter your topic name. No login.

## Tuning

Edit `monitor/config.yaml`:

- `cities` — add/remove EMEA hubs. Each entry needs `name` (used in the
  search bucket name), `location` (the free-text string passed to JobSpy),
  and `country_indeed` (a country name from `jobspy.model.Country`).
- `role_templates` — each template becomes a search bucket per city. Add
  more `search_terms` to broaden coverage; they all upsert into the same
  bucket so you don't get duplicate notifications.
- `filters.include_companies` — the allowlist. Synonyms can be nested as
  inner lists (e.g. `[tiktok, bytedance]`). Matching is case-insensitive
  substring — "google" matches "Google LLC".
- `filters.exclude_titles` — case-insensitive substring on the title. The
  pre-populated set catches senior / staff / lead / level-5+ wording.
- `filters.min_description_chars` — drops suspiciously empty postings.

## Inspect the DB locally

```sh
sqlite3 monitor/jobs.db
sqlite> .schema
sqlite> SELECT count(*) FROM jobs WHERE status = 'active';
sqlite> SELECT title, company, location, first_seen
        FROM jobs ORDER BY first_seen DESC LIMIT 20;
sqlite> SELECT search_name, sum(rows_scraped), sum(rows_new)
        FROM runs WHERE started_at > date('now','-7 day')
        GROUP BY search_name ORDER BY 3 DESC;
```

## Run locally

Run from your home IP (LinkedIn / Glassdoor / Bayt block data-center IPs):

```sh
pip install -e .
pip install -r monitor/requirements.txt
export NTFY_TOPIC=your-topic-here   # (PowerShell: $env:NTFY_TOPIC = "...")
python -m monitor.run
```

This appends to the same `jobs.db`, so each run dedupes against the last one.
Commit `monitor/jobs.db` and the rendered markdown afterwards if you want the
GitHub copies to stay current.

## What to expect

- **`JOBS.md`** at the repo root renders the current active jobs as a
  GitHub-friendly table grouped into FAANG+ / Quant / Other tiers, with
  per-row Apply buttons. It overwrites every run so age counters tick up
  and gone roles disappear.
- Zero net-new jobs = no ntfy notification (no spam).
- Logs land in `monitor/logs/` (gitignored) — check there when a run fails.
