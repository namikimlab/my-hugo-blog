---
title: "💯 From Basic to Intermediate: Understanding dbt Tests"
date: 2025-06-10
tags: ["dbt", "data-engineering", "sql", "testing"]
categories: ["Data Engineering"]
---

If you're using **dbt** to transform data, you're already winning.  

But did you know dbt has powerful testing features to keep your data clean, reliable, and trustworthy?

In this post, we’ll walk through:

- ✅ **Basic dbt tests** — the quick wins
- 🚀 **Intermediate tests** — custom logic and reusable macros


## ✅ Basic dbt Tests (Built-in)

dbt has **out-of-the-box tests** you can define in your `.yml` files under your models. Here’s an example:

```yaml
version: 2

models:
  - name: customers
    description: Customer master table
    columns:
      - name: customer_id
        tests:
          - not_null
          - unique
      - name: email
        tests:
          - not_null
````

### 🔧 What these do:

* `not_null`: ensures no NULLs in the column
* `unique`: makes sure values are unique
* `accepted_values`: (optional) lets you whitelist values
* `relationships`: checks if foreign keys match the referenced table

### Example: `accepted_values`

```yaml
      - name: status
        tests:
          - accepted_values:
              values: ['active', 'inactive', 'suspended']
```

These basic tests are easy to add and powerful for catching simple data issues — **before they hit production dashboards**.


## 🚀 Intermediate dbt Tests (Custom + Reusable)

Sometimes, built-in tests aren’t enough. You need logic like:

* Check for duplicate user records **within a date range**
* Validate order total = sum of line items
* Ensure one active subscription per user

That’s where **custom SQL-based tests** and **macros** come in.

### 🧪 Example: Custom SQL Test

Create a file in `tests/`:

```sql
-- tests/one_active_subscription.sql

SELECT user_id
FROM {{ ref('subscriptions') }}
GROUP BY user_id
HAVING SUM(CASE WHEN status = 'active' THEN 1 ELSE 0 END) > 1
```

Then reference it in your `.yml`:

```yaml
      - name: user_id
        tests:
          - one_active_subscription
```

Boom! dbt will fail the build if a user has multiple active subs.


### 🧰 Bonus: Reusable Macros for Parametrized Tests

Say you want to check if a numeric column is **always positive**, across multiple models.

Create a macro in Jinja:

```jinja
-- macros/test_positive_values.sql

{% test positive_values(model, column_name) %}
SELECT *
FROM {{ model }}
WHERE {{ column_name }} < 0
{% endtest %}
```

Then use it in `.yml`:

```yaml
      - name: price
        tests:
          - positive_values
```

Now you have reusable logic that works like a built-in test!

### Difference between dbt custom test and macro

A dbt custom SQL test is a one-off .sql file that contains a specific query to check a condition — it’s great for unique cases tied to a single model. In contrast, a reusable macro test is written once as a Jinja macro and can be applied across multiple models or columns with parameters, making it more flexible and DRY (Don't Repeat Yourself). Use custom SQL for specific edge cases, and macros when you need consistency across your data tests.
## 🧵 Final Thoughts

| Level        | Tooling Used        | Examples                                     |
| ------------ | ------------------- | -------------------------------------------- |
| Basic        | Built-in YAML tests | `not_null`, `unique`, `accepted_values`      |
| Intermediate | Custom SQL/macros   | `positive_values`, `one_active_subscription` |

Start small. Add `not_null` and `unique` everywhere.
Then gradually level up with custom logic where needed.

Your future self (and your data team) will thank you.
And remember: **"Broken data pipelines never lie — they just test your patience."**

Happy testing with dbt! 🧪
