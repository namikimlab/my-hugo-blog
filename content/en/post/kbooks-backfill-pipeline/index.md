+++
title = "📚 Building a 25-Year Backfill Pipeline for the National Library of Korea API"
date = 2025-10-22T12:00:00+09:00
tags = ["data engineering", "python", "systemd", "postgresql", "etl", "pipeline", "api"]
categories = ["Data Engineering"]
draft = false
+++

### How I Designed a Reliable, Auto-Resuming ETL to Collect Decades of Book Data — Without Airflow

## 1. Why I Built This

The **National Library of Korea (NLK)** provides a public API called *Seoji* — a bibliographic catalog of all registered books in Korea.

I wanted to collect **the entire dataset**, from **January 2000 to December 2024**, and store it in my PostgreSQL database (Supabase).

It sounded simple at first — just a loop over API pages.
But in practice, I had to solve:

* Rate limits and connection timeouts,
* 25 years × 12 months of data (300+ runs),
* Occasional EC2 disconnects,
* Safe deduplication for millions of JSON records.

## 2. Understanding the NLK API

Reference: [NLK Seoji API Docs](https://www.nl.go.kr/NL/contents/N31101010000.do)

Key constraints I found early:

* Must iterate in small chunks, `page_size = 100`.
* Requires an API key (`cert_key`).
* Sort options limited to `INPUT_DATE` or `PUBLISH_PREDATE`.

That meant I couldn’t pull everything at once.
I decided to **partition by month**, fetching one month at a time with clear start and end dates.


## 3. The First Prototype — A Simple Fetcher

I began with `fetch_pages.py`, which:

* Pulled pages sequentially,
* Inserted JSON into `raw_nl_books`,
* Logged progress to the console.

It *worked*, but it wasn’t sustainable:

* No resume logic — if the job crashed, I lost progress.
* Couldn’t survive an EC2 reboot.
* I’d have to manually change the month each time.

## 4. From Manual to Autonomous

I needed the system to:

1. **Fetch all months automatically**, from 2000 → 2024.
2. **Recover from network drops or timeouts**.
3. **Avoid duplicates** even if re-run.
4. Run safely for days without supervision.

That meant I needed something more resilient — but still lightweight.

## 5. Orchestration Options I Considered

| Option               | Pros                                     | Cons                                 | Decision     |
| -------------------- | ---------------------------------------- | ------------------------------------ | ------------ |
| **Cron**             | Easy to schedule                         | No retry or backoff; hard to monitor | ❌            |
| **Airflow / Kestra** | Full orchestration, UI                   | Overkill for one sequential task     | ❌            |
| **systemd**          | Built-in restart, logs, no external deps | Linux-only                           | ✅ **Chosen** |

I realized **systemd** already gave me what I needed:

* Auto-restart on failure,
* Persistent logs (`journalctl`),
* Easy enable/disable,
* Zero setup beyond one service file.

## 6. Designing a Checkpoint System

I created a simple folder:
`~/nlk-state/`

Each month has two small files:

```
2000-01.page   # last fetched page
2000-01.done   # month completed
```

If the system crashes, the next run just reads `.page` and resumes that month.

**Why file-based checkpoints?**

* No DB lockups.
* Transparent: I can open and see progress.
* Works even if DB is temporarily unavailable.

## 7. The Monthly Runner

I extended `fetch_pages_month.py` to:

* Fetch a specific month window (`start_publish_date` → `end_publish_date`).
* Write `.page` after each success.
* Exit non-zero on timeout (so systemd restarts it).
* Use `rec_hash = md5(source_record::text)` for idempotent inserts.

Each month now runs cleanly and stops when NLK returns zero results.


## 8. The Master Manager: `run_all_months.py`

This script drives the entire 25-year loop:

```text
2000-01 → 2000-02 → … → 2024-12
```

For each month:

* If `.done` exists → skip.
* Else → run `fetch_pages_month.py` as a subprocess.
* On success → create `.done` and move on.
* On error → exit (systemd restarts it and resumes the same month).

### Why a “manager” instead of a single giant loop?

Because this keeps boundaries clear:

* The month script focuses on API + DB work.
* The manager focuses on flow control and checkpointing.

## 9. Supervisor: `systemd`

My `nlk-history.service` file runs the manager in the background.

```ini
[Service]
User=ec2-user
WorkingDirectory=/home/ec2-user/kbook-data-pipeline
ExecStart=/home/ec2-user/kbook-data-pipeline/venv/bin/python /home/ec2-user/kbook-data-pipeline/scripts/run_all_months.py
Restart=on-failure
RestartSec=30s
```

**Why systemd over Airflow or Cron?**

* Restarts automatically on non-zero exit.
* Doesn’t require any UI or database.
* Clean logs via `journalctl`.
* Perfect for long EC2 jobs.

When the final month finishes, the script exits 0 and systemd stops gracefully.


## 10. Verifying the Pipeline

**Sanity checks:**

* `.page` updates every few minutes.
* `.done` appears when each month completes.
* Restarted EC2 → resumed same page automatically.
* No duplicate inserts (thanks to `rec_hash` uniqueness).

Full runtime:
≈ 30 minutes × 300 months ≈ 6 days of continuous crawling.


## 11. Challenges Solved

| Problem         | Solution                                                        |
| --------------- | --------------------------------------------------------------- |
| Duplicate IDs   | Reseeded identity column (`ALTER SEQUENCE … RESTART`)           |
| Duplicate rows  | Added `rec_hash` as generated unique column                     |
| Timeout errors  | Added exponential backoff + longer read timeout                 |
| EC2 disconnects | Moved job under `systemd` supervision                           |


## 12. Results

After a week of unattended runtime:

* ✅ Full dataset from 2000–2024 stored in Postgres
* ✅ Zero duplicate rows
* ✅ Automatic restarts during network hiccups
* ✅ Completely hands-off process

In total: **~6 million book records** collected and stored safely.


## 13. Lessons Learned

1. **Simplicity scales** — small, reliable scripts outperform heavy frameworks when you only need sequential control.
2. **Explicit checkpoints > retries.**
3. **systemd** is underrated as a production-grade job runner.
4. Design for **resumability first**, optimization later.
