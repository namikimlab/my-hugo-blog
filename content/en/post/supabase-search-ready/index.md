---
title: "Making Postgres Search Fast and Accurate with FTS and dbt Indexes"
date: 2025-10-23
tags: ["postgres", "supabase", "dbt", "full-text-search", "data-engineering", "indexing"]
categories: ["Data Engineering"]
---

Here's how I added search to my **KBooks** site. 

All my book data was already flowing into **Supabase** from the National Library of Korea API through a dbt pipeline, so the challenge was:
**How do I make this table searchable efficiently without adding another service at this stage?**

This is how I got database-level search ready, step by step, and why each decision made sense.

## 1. Start from a clean source: `silver_books`

My raw data lands as JSON in `raw_nl_books`.
From there, I built a `silver_books` table using dbt. Each row represents one unique book identified by its **ISBN-13**, and only valid ISBNs are kept.

That table already had the core fields a user might search:
`title`, `author`, `publisher_name`, `subject`, and `book_introduction`.

So the obvious thought was: *“Let’s just add search on top of this table.”*

## 2. Understand what “search” means in Postgres

Postgres offers two main ways to search text:

### 🔍 Full-Text Search (FTS)

FTS tokenizes text into words (like “Harry” and “Potter”) and matches them intelligently, not by raw string comparison.
It can even score results by **relevance** (`ts_rank`), so the most likely matches come first.

### ✏️ Trigram search

A trigram is a three-character chunk of text.
Trigram indexes let Postgres find words even with typos or partial matches—useful for languages like Korean or for fuzzy matching (for example, “해리포” still finds “해리 포터”).

These two together cover both *exact* and *fuzzy* cases without needing an external search engine.


## 3. Decide what to index (and what not to)

Originally I included **every text field** (title, author, publisher, subject, introduction) in search.
It worked—but it brought in irrelevant results.
For example, a book would match just because “해리 포터” appeared once in its *introduction* field.

So I simplified:

* **Include:** `title`, `author`
* **Exclude:** `subject`, `book_introduction`, `publisher_name`

That change instantly made results cleaner and more relevant—exactly what a user expects.

## 4. Build it with dbt

I used dbt to handle both transformation and index creation in one place.

* **FTS column:**
  I added a computed column `tsv = to_tsvector('simple', title || ' ' || author)`
  This creates a compressed summary of searchable words.

* **Indexes (via dbt `post_hook`):**

  1. `unique(isbn13)` → primary key for deduplication
  2. `GIN(tsv)` → for full-text search
  3. `GIN(title gin_trgm_ops)` → for fuzzy title matches

dbt automatically runs these commands after it rebuilds the table, so the indexes are always fresh.


## 5. Sanity-check the results

I tested with a simple query:

```
select
  isbn13, title, author_display,
  ts_rank(tsv, plainto_tsquery('simple', '해리 포터')) as rank
from kbooks_dbt.silver_books
where tsv @@ plainto_tsquery('simple', '해리 포터')
order by rank desc
limit 20;
```

The output looked perfect—only actual *Harry Potter* titles showed up, sorted by relevance.
No random books just because they mentioned him in the description.


## 6. Why this architecture makes sense

| Goal           | Choice                            | Reason                                            |
| -------------- | --------------------------------- | ------------------------------------------------- |
| Fast search    | GIN indexes                       | O(1) lookups instead of scanning millions of rows |
| Typos & Korean | Trigram                           | Handles partial & spaced words                    |
| Simplicity     | Keep search inside `silver_books` | No extra tables or sync jobs                      |
| Clean results  | Title + Author only               | Filters noise from other fields                   |
| Maintainable   | dbt builds & indexes together     | One source of truth                               |

This setup will easily scale to millions of rows and is trivial to extend later.
If I ever need ranking pages, analytics, or external search (like Meilisearch), I can just export from `silver_books` without changing the web app logic.


## 7. The result

Now, a user can type **“해리 포터”** on the KBooks website and get instant, relevant results straight from Supabase.
No third-party search engine, no sync delay—just well-indexed SQL doing its job.


## 8. Key takeaway

**Good search doesn’t require a search engine.**
If your data already lives in Postgres, learning to use `to_tsvector` and GIN indexes can get you 80% of the way there—with no new infrastructure.

Building this directly into the dbt pipeline also means:

* search always works on the latest cleaned data,
* all transformations and performance optimizations live in one version-controlled repo.

That’s exactly what I wanted for KBooks: clean, minimal, and fast.

