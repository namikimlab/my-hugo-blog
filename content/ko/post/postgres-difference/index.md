---
title: "PostgreSQL의 의외의 함정: Boolean, 텍스트 I/O, 그리고 ETL 이슈"
date: 2025-06-10
tags: ["postgresql", "etl", "boolean", "dbt", "data-engineering"]
categories: ["Data Engineering"]
---

PostgreSQL은 강력하고 표준을 잘 따르는 데이터베이스입니다.  
하지만 의외의 **작은 함정들**도 있죠.  

그중 하나는 바로 **boolean 값을 다루는 방식**, 특히 데이터를 텍스트 형식으로 내보낼 때의 이야기입니다.


## 🧠 PostgreSQL의 Boolean 처리 방식: 생각과 다르다

PostgreSQL은 `boolean` 값을 내부적으로 **1비트(bit)** 만으로 효율적으로 저장합니다. 예상한 대로죠.

하지만 그 값을 **텍스트**로 변환하거나 `COPY` 같은 방식으로 내보내면, 결과는 좀 다릅니다:

```sql
SELECT true::text;   -- 결과: 't'
SELECT false::text;  -- 결과: 'f'
````

맞습니다 — **`true`는 `'t'`**, **`false`는 `'f'`** 로 표현됩니다.

이건 PostgreSQL의 **텍스트 I/O 기본 동작 방식**인데,
이 동작 때문에 시스템 간 데이터를 주고받을 때 미묘한 버그가 생기기도 합니다.


## 🗂️ COPY & Export 이슈

`COPY TO`, `psql`, `pg_dump` 혹은 `airflow`, `dbt`, `pandas` 같은 도구를 쓸 때
PostgreSQL은 boolean 값을 `'t'` / `'f'` 형태로 내보냅니다 — `true` / `false`가 아닙니다!

예를 들어:

```sql
COPY my_table TO '/tmp/export.csv' CSV HEADER;
```

이렇게 내보내면 boolean 컬럼은 다음과 같이 나옵니다:

```csv
id,is_active
1,t
2,f
```

만약 이 데이터를 받는 도구가 `true/false` 또는 `1/0`을 기대하고 있다면...
결과는 혼돈의 카오스입니다. 🌀


## 🧪 요약: 저장 방식 vs 표현 방식

| 계층                   | PostgreSQL의 실제 동작             |
| -------------------- | ----------------------------- |
| **내부 저장소**           | 1비트 (0 또는 1)                  |
| **텍스트 캐스팅**          | `'t'` / `'f'`                 |
| **COPY/psql export** | `'t'` / `'f'`                 |
| **CSV 파서**           | 보통 `true/false` 또는 `1/0`을 기대함 |


## 💡 실무 팁

ETL 파이프라인 (dbt, pandas, Airflow 등)에서 `'t'`, `'f'`를 발견했다면:

✅ 당황하지 마세요 — PostgreSQL에선 정상입니다
✅ export/import 시 boolean 형식을 명시적으로 변환하세요
✅ downstream 도구들이 어떤 형식을 기대하는지 문서화하세요


## 🧵 마무리

PostgreSQL은 정말 훌륭한 데이터베이스지만,
**MySQL, BigQuery, Pandas**처럼 동작하진 않습니다.

Boolean 관련 이 작은 디테일 하나만 봐도
**DB의 I/O 동작을 제대로 아는 것만으로도 디버깅 시간을 크게 줄일 수 있다는 것**을 알 수 있죠.

> "버그가 아니라, PostgreSQL은 그냥 PostgreSQL일 뿐."

