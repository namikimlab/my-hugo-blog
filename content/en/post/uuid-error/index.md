---
title: "UUID Pitfalls in Spark → Kafka → Postgres Pipelines"
date: 2025-06-07
tags: ["spark", "kafka", "postgres", "uuid", "data-engineering", "streaming"]
categories: ["Data Engineering"]
---

I was building a data pipeline using Kafka and Spark structured streaming. Fully containerized. 

The stack:

- **Kafka** for streaming transaction data  
- **Spark Structured Streaming** for real-time processing and fraud detection  
- **Postgres** as the data warehouse

Everything was smooth. Until one tiny villain showed up:

> UUID fields.

Yes — UUIDs. 

Here’s exactly what happened (so you can avoid the same headache).


## ✅ The Original Design 
I designed the tables in Postgres like this:

```sql
CREATE TABLE fact_transaction (
    transaction_id UUID PRIMARY KEY,
    customer_id UUID REFERENCES dim_customer(customer_id),
    merchant_id UUID REFERENCES dim_merchant(merchant_id),
    ...
);
````

Kafka was happily publishing events with UUIDs serialized as strings (since JSON doesn’t do native UUIDs).

Spark was happily reading them using:

```python
schema = StructType([
    StructField("transaction_id", StringType()),
    StructField("customer_id", StringType()),
    StructField("merchant_id", StringType()),
    ...
])
```

All green lights. Until...



## ❌ The "WHY IS THIS FAILING?!" Moment

As Spark tried to write into Postgres via JDBC, kaboom:

```
ERROR: column "transaction_id" is of type uuid but expression is of type character varying
Hint: You will need to rewrite or cast the expression.
```

But wait… I *am* passing valid UUID strings!


## 🔬 Where It Actually Broke (Spoiler: JDBC Shenanigans)

Here’s what was really happening under the hood:

* Spark reads UUIDs as strings (`StringType()`)
* JDBC sends them to Postgres as `VARCHAR`
* Postgres (strict little thing) says:

  > “I'm not auto-casting VARCHAR to UUID when you’re using prepared statements.”

In plain SQL, Postgres might cast for you.
In JDBC prepared statements? Nope. Strict mode activated.

And to make things worse: Spark doesn’t even have a native `UUIDType()` — you're stuck with `StringType()`.


## 🚧 False Solutions I Tried (Don't Do This)

I tried all sorts of hacks:

* **Cast UUIDs again in Spark?** → Made no difference.
* **Play with JDBC options like `stringtype=unspecified`?** → Unreliable.
* **Write ugly casting logic in Spark SQL?** → Works today, breaks tomorrow.

Nothing worked cleanly.


## 🔨 The Real Fix (Simple and Effective)

Finally, I stopped fighting and embraced reality:

> **Store UUIDs as `TEXT` in Postgres.**

```sql
ALTER TABLE fact_transaction ALTER COLUMN transaction_id TYPE TEXT USING transaction_id::TEXT;
```

I updated both fact and dimension tables to store UUIDs as text.

Why? Because:

* Kafka produces UUIDs as strings ✅
* Spark reads them as `StringType()` ✅
* JDBC sends them as `VARCHAR` ✅
* Postgres accepts `TEXT` ✅

Everything finally *just worked*.


## 🧠 Industry Secret: TEXT is Totally Fine

Here’s the thing most people won’t tell you:

* Snowflake, BigQuery, Redshift — no native UUID types.
* BI tools? They love strings.
* Debugging? Easier with TEXT.
* Schema evolution? Smoother.

Unless you’re building OLTP-level transactional systems, TEXT for UUIDs is often the more *pragmatic* choice in data pipelines.


## 🔄 The Final Architecture 

* Kafka → Spark Streaming → Postgres
* No more casting hacks
* Fully containerized via Docker Compose
* Automated dimension seeding
* Just `docker compose up` — and relax


## 🚀 Lessons Learned

* UUID + Spark + JDBC + Postgres = ticking time bomb ⚠️
* TEXT > UUID for analytical pipelines (most of the time)
* Simplify your type system across the stack
* Optimize for **operational smoothness**, not theoretical purity


At the end of the day:  
**Real-world data engineering = hitting weird walls.**  
**Surviving them = becoming a better engineer.**

