---
title: "How PostgreSQL Surprises You: Booleans, Text I/O, and ETL Gotchas"
date: 2025-06-10
tags: ["postgresql", "etl", "boolean", "dbt", "data-engineering"]
categories: ["Data Engineering"]
---

PostgreSQL is a powerful, standards-compliant database — but it has its quirks.  

One of those is **how it handles boolean values**, especially when exporting data in text format.


## 🧠 PostgreSQL Boolean Behavior: It's Not What You Think

Internally, PostgreSQL stores `boolean` values efficiently using **just 1 bit** — as you'd expect.

But when you convert those values to **text**, say in a query or an export via `COPY`, things look… different:

```sql
SELECT true::text;   -- returns 't'
SELECT false::text;  -- returns 'f'
````

Yep — **`true` becomes `'t'`** and **`false` becomes `'f'`**.

That’s PostgreSQL’s default behavior in **text I/O** — and it can cause subtle bugs when moving data between systems.


## 🗂️ COPY & Export Gotchas

When using `COPY TO` (or tools like `psql`, `pg_dump`, or `airflow`/`dbt`/`pandas` under the hood), PostgreSQL exports booleans as `'t'`/`'f'`, not `true`/`false`.

So this:

```sql
COPY my_table TO '/tmp/export.csv' CSV HEADER;
```

…will export boolean columns like:

```csv
id,is_active
1,t
2,f
```

If your downstream tools expect `true/false` or `1/0`, you're in for a confusing time. 🌀


## 🧪 Summary: Storage vs Representation

| Layer                | What PostgreSQL Actually Does      |
| -------------------- | ---------------------------------- |
| **Internal storage** | 1 bit (0 or 1)                     |
| **Text casting**     | `'t'` / `'f'`                      |
| **COPY/psql export** | `'t'` / `'f'`                      |
| **CSV readers**      | Often expect `true/false` or `1/0` |


## 💡 Real-World Tip

If you're working with ETL pipelines (dbt, pandas, Airflow, etc.) and you see `'t'` and `'f'` in your data:

✅ Don’t panic — it's normal in PostgreSQL.
✅ Explicitly cast or transform booleans when exporting/importing.
✅ Document the expected format for downstream tools.

## 🧵 Closing Thought

PostgreSQL is amazing, but it **doesn't always behave like MySQL, BigQuery, or Pandas**.

This tiny detail about booleans is a great example of why **knowing your database’s I/O behaviors** can save hours of debugging.

> "It's not a bug — it's just PostgreSQL being PostgreSQL."
