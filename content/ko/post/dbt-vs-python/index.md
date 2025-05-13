+++
title = "📊 dbt가 잘하는 일 vs Python이 잘하는 일"
date = 2025-05-12T12:00:00+09:00
tags = ["data engineering", "python", "dbt"]
categories: ["Data Engineering"]
draft = false
+++

| 역할 | dbt가 잘함 | Python이 더 낫다 |
|------|------------|------------------|
| 정형 데이터 정제 (staging) | ✅ | 가능은 하지만 불편함 |
| 마트 테이블 구조 설계 | ✅ | 가능은 함 |
| 사용자별로 달라지는 계산 | ❌ 불편함 | ✅ 매우 유연함 |
| 점수화, 조건 매칭, if-else 로직 | ❌ 매우 번거로움 | ✅ 적합 |
| 사용자 입력 기반 필터링 | ❌ 불가능 | ✅ 핵심 기능 |
| 추천 이유 설명, 로직 튜닝 | ❌ | ✅ 완전 맞춤형 구현 가능 |

## 예를 들어
```sql
-- dbt에서는 이런 로직이 아주 힘들다...
SELECT
  CASE 
    WHEN user.age BETWEEN policy.min_age AND policy.max_age THEN 30
    ELSE 0
  END +
  CASE 
    WHEN user.income < policy.income_ceiling THEN 25
    ELSE 0
  END + ...
```

- dbt에서는 “user”란 존재 자체가 없음  
- dbt는 “모든 사용자에게 동일하게 적용되는 모델”을 설계하는 도구  
- 반면 Python에서는 사용자가 입력할 때마다 추천 결과가 달라지게 만들 수 있음

> 👉 **dbt는 정적(Static) 모델링에 적합하지만 사용자 입력 기반의 동적(Dynamic) 추천 시스템은 Python이 더 났다.**

## 상황 가정
API로 받은 정책 데이터 예시 (JSON or CSV로 저장됨)
```json
{
  "policy_name": "청년 월세 지원",
  "eligibility": "만 19세 이상 34세 이하 / 무직 또는 재직 / 독립가구",
  "target_region": "전국",
  "link": "https://example.com"
}
```
조건이 텍스트로 되어있다.   
추천 점수 계산을 하려면 숫자나 값으로 된 컬럼이 필요하다.

## 그래서 필요한 게 바로 dbt 구조화!
- "만 19세 이상 34세 이하" → min_age = 19, max_age = 34
- "무직 또는 재직" → job_status = '무직,재직'
- "독립가구" → household_type = '독립가구'
- 그리고 target_region, income_ceiling 등등도 컬럼화

## dbt 구조화 흐름

### 1. raw_policies 테이블 (API 받아서 저장한 그대로)
```sql
SELECT * FROM {{ source('raw', 'policies') }}
```

## 2. stg_policies.sql (정제 단계)
```sql
SELECT
  policy_name,
  REGEXP_EXTRACT(eligibility, '만 (\d{2})세 이상')::INT AS min_age,
  REGEXP_EXTRACT(eligibility, '(\d{2})세 이하')::INT AS max_age,
  CASE
    WHEN eligibility LIKE '%무직%' AND eligibility LIKE '%재직%' THEN '무직,재직'
    WHEN eligibility LIKE '%무직%' THEN '무직'
    WHEN eligibility LIKE '%재직%' THEN '재직'
    ELSE '무관'
  END AS job_status,
  CASE
    WHEN eligibility LIKE '%독립가구%' THEN '독립가구'
    ELSE '무관'
  END AS household_type,
  target_region,
  link
FROM {{ source('raw', 'policies') }}
```

## 3. mart_policies.sql (추천 엔진이 사용할 최종 테이블)
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

## 결론
- API로 받은 텍스트 기반 정책 조건을 → 정제된 수치/값 기반 테이블로 만드는 게 dbt 구조화  
- 그래야 Python 추천 로직에서 조건 필터링/점수 계산을 자동화할 수 있다.
