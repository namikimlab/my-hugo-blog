---
title: "💯 dbt 테스트 이해하기: 기본부터 중급까지"
date: 2025-06-10
tags: ["dbt", "data-engineering", "sql", "testing"]
categories: ["Data Engineering"]
---

**dbt**로 데이터를 변환하고 있다면 이미 잘하고 있는 겁니다. 🙌  

그런데 dbt에는 **데이터를 더 깨끗하고 신뢰할 수 있게 유지하는 강력한 테스트 기능**도 있다는 거, 알고 계셨나요?

이번 글에서는 다음을 다룹니다:

- ✅ **기본 dbt 테스트** — 빠르게 적용 가능한 기본기
- 🚀 **중급 테스트** — 사용자 정의 로직과 재사용 가능한 매크로



## ✅ 기본 dbt 테스트 (Built-in)

dbt는 모델의 `.yml` 파일 안에서 바로 사용할 수 있는 **기본 테스트**들을 제공합니다. 예시:

```yaml
version: 2

models:
  - name: customers
    description: 고객 마스터 테이블
    columns:
      - name: customer_id
        tests:
          - not_null
          - unique
      - name: email
        tests:
          - not_null
````

### 🔧 각 테스트가 하는 일:

* `not_null`: 컬럼에 NULL 값이 없는지 확인
* `unique`: 값이 고유한지 검증
* `accepted_values`: 허용된 값만 포함되어 있는지 체크
* `relationships`: 외래 키가 참조 대상 테이블과 매칭되는지 확인

### `accepted_values` 예시

```yaml
      - name: status
        tests:
          - accepted_values:
              values: ['active', 'inactive', 'suspended']
```

이런 기본 테스트들은 **생산 대시보드에 오류가 반영되기 전에** 단순한 데이터 이상을 빠르게 잡을 수 있습니다.



## 🚀 중급 dbt 테스트 (사용자 정의 + 재사용 가능)

기본 테스트만으로는 부족할 때가 있습니다. 예를 들어:

* 특정 기간 내 중복 사용자 존재 여부 확인
* 주문 총액이 각 항목의 합과 일치하는지 검증
* 사용자당 하나의 활성 구독만 허용

이럴 때는 **사용자 정의 SQL 테스트**와 **매크로**를 활용하면 됩니다.

### 🧪 사용자 정의 SQL 테스트 예시

`tests/` 폴더에 다음과 같이 작성합니다:

```sql
-- tests/one_active_subscription.sql

SELECT user_id
FROM {{ ref('subscriptions') }}
GROUP BY user_id
HAVING SUM(CASE WHEN status = 'active' THEN 1 ELSE 0 END) > 1
```

그리고 `.yml`에서 호출:

```yaml
      - name: user_id
        tests:
          - one_active_subscription
```

활성 구독이 2개 이상인 유저가 있다면 테스트가 실패하게 됩니다.

---

### 🧰 보너스: 파라미터화된 재사용 가능한 매크로 테스트

여러 모델에서 **숫자 컬럼이 항상 양수인지** 검증하고 싶다면?

Jinja 매크로로 다음과 같이 작성:

```jinja
-- macros/test_positive_values.sql

{% test positive_values(model, column_name) %}
SELECT *
FROM {{ model }}
WHERE {{ column_name }} < 0
{% endtest %}
```

그리고 `.yml`에서 사용:

```yaml
      - name: price
        tests:
          - positive_values
```

이제 한 번 정의한 테스트 로직을 여러 모델/컬럼에 재사용할 수 있게 됩니다.

### dbt 커스텀 테스트와 매크로 차이 

**dbt 사용자 정의 SQL 테스트**는 특정 모델에 특화된 조건을 확인하는 단일 `.sql` 파일입니다.
반면에 **재사용 가능한 매크로 테스트**는 Jinja 매크로로 작성되며, 여러 모델/컬럼에 파라미터로 적용할 수 있어 더 유연하고 \*\*DRY(Don’t Repeat Yourself)\*\*합니다.
**특정한 케이스에는 SQL 테스트**, **여러 곳에 공통으로 적용할 땐 매크로 테스트**를 쓰는 게 좋습니다.


## 🧵 마무리 정리

| 수준 | 사용하는 도구        | 예시                                           |
| -- | -------------- | -------------------------------------------- |
| 기본 | 내장 YAML 테스트    | `not_null`, `unique`, `accepted_values`      |
| 중급 | 사용자 정의 SQL/매크로 | `positive_values`, `one_active_subscription` |

처음엔 작게 시작하세요. `not_null`과 `unique`만 적용해도 큰 차이를 만듭니다.
그리고 점점 복잡한 테스트 로직을 추가해보세요.

미래의 당신과 팀원들이 **진짜로 감사할 일**입니다.

**“망가진 데이터 파이프라인은 거짓말하지 않는다. 단지 당신의 인내심을 시험할 뿐이다.”**

dbt로 즐거운 데이터 테스트 되세요! 🧪

