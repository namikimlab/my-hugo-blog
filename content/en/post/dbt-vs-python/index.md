+++
title = "📊 What dbt Does Well vs What Python Does Better"
date = 2025-05-12T12:00:00+09:00
draft = false
+++

| Role | dbt Does Well | Python Does Better |
|------|----------------|---------------------|
| Structured data cleaning (staging) | ✅ | Possible, but inconvenient |
| Designing mart table structures | ✅ | Also possible |
| User-specific calculations | ❌ Inconvenient | ✅ Super flexible |
| Scoring, conditional matching, if-else logic | ❌ Very cumbersome | ✅ Ideal |
| Filtering based on user input | ❌ Not possible | ✅ Core feature |
| Explaining recommendations, tuning logic | ❌ | ✅ Fully customizable |

## For Example

```sql
-- This kind of logic is painful in dbt...
SELECT
  CASE 
    WHEN user.age BETWEEN policy.min_age AND policy.max_age THEN 30
    ELSE 0
  END +
  CASE 
    WHEN user.income < policy.income_ceiling THE_
    ELSE 0
  END + ...
```
- In dbt, the concept of a “user” doesn’t even exist
- dbt is built for models that apply the same logic to everyone
- Python, on the other hand, can generate different recommendations per user based on input

> **👉 dbt is great for static modeling, but dynamic, user-input-driven recommender systems are better suited for Python.**

## Use Case 
Sample policy data received from an API (stored as JSON or CSV):
```json
{
  "policy_name": "Youth Monthly Rent Support",
  "eligibility": "Ages 19 to 34 / Unemployed or Employed / Independent household",
  "target_region": "Nationwide",
  "link": "https://example.com"
}
```
The conditions are written in natural language.
To calculate recommendation scores, we need columns with actual values.

## That’s where dbt’s structuring comes in!
- "Ages 19 to 34" → min_age = 19, max_age = 34  
- "Unemployed or Employed" → job_status = 'Unemployed,Employed'  
- "Independent household" → household_type = 'Independent'  
- And also turn target_region, income_ceiling, etc. into proper columns  

## dbt Structuring Workflow
### 1. raw_policies table (raw data straight from API)
```sql
SELECT * FROM {{ source('raw', 'policies') }}
```

### 2. stg_policies.sql (staging layer)
```sql
SELECT
  policy_name,
  REGEXP_EXTRACT(eligibility, 'Ages (\d{2})')::INT AS min_age,
  REGEXP_EXTRACT(eligibility, 'to (\d{2})')::INT AS max_age,
  CASE
    WHEN eligibility LIKE '%Unemployed%' AND eligibility LIKE '%Employed%' THEN 'Unemployed,Employed'
    WHEN eligibility LIKE '%Unemployed%' THEN 'Unemployed'
    WHEN eligibility LIKE '%Employed%' THEN 'Employed'
    ELSE 'Not specified'
  END AS job_status,
  CASE
    WHEN eligibility LIKE '%Independent household%' THEN 'Independent'
    ELSE 'Not specified'
  END AS household_type,
  target_region,
  link
FROM {{ source('raw', 'policies') }}
```

### 3. mart_policies.sql (final table used by recommender engine)
```sql
SELECT
  policy_name,
  min_age,
  max_age,
  job_status,
  household_type,
  target_region,
  link
FROM {{ ref('stg_policies') }}
WHERE min_age IS NOT NULL AND max_age IS NOT NULL
```

## In Conclusion
dbt helps convert messy, text-based policy conditions from the API into clean, structured, value-based tables.
That’s what makes it possible for Python’s recommender logic to automatically filter conditions and compute scores.
