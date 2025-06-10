---
title: "Spark → Kafka → Postgres 파이프라인에서 UUID가 터트린 지뢰"
date: 2025-06-07
tags: ["spark", "kafka", "postgres", "uuid", "data-engineering", "streaming"]
categories: ["Data Engineering"]
---

Kafka와 Spark Structured Streaming을 이용해서 데이터 파이프라인을 구축하고 있었습니다. 완전히 컨테이너화된 시스템.  

스택 구성은 이렇습니다:

- **Kafka** → 거래 데이터를 스트리밍으로 전송  
- **Spark Structured Streaming** → 실시간 처리 및 이상 거래 탐지  
- **Postgres** → 데이터 웨어하우스

모든 게 순조로웠습니다. 그런데 갑자기 등장한 한 놈.

> UUID 필드.

맞습니다 — UUID.  

이제 어떤 일이 벌어졌는지 정확히 보여드릴게요. 

## ✅ 원래 설계

Postgres 테이블을 이렇게 설계했죠:

```sql
CREATE TABLE fact_transaction (
    transaction_id UUID PRIMARY KEY,
    customer_id UUID REFERENCES dim_customer(customer_id),
    merchant_id UUID REFERENCES dim_merchant(merchant_id),
    ...
);
````

Kafka는 UUID를 문자열로 직렬화해서 이벤트를 잘 뿌려주고 있었습니다 (JSON은 원래 UUID 타입이 없으니까요).

Spark도 문제없이 읽어왔습니다:

```python
schema = StructType([
    StructField("transaction_id", StringType()),
    StructField("customer_id", StringType()),
    StructField("merchant_id", StringType()),
    ...
])
```

모든 게 초록불이었죠.
그런데…


## ❌ "왜 안돼?!" 순간

Spark가 Postgres에 JDBC로 데이터를 쓰려고 하자마자 바로 폭발:

```
ERROR: column "transaction_id" is of type uuid but expression is of type character varying
Hint: You will need to rewrite or cast the expression.
```

잠깐… 나 유효한 UUID 문자열 넘기고 있는데??


## 🔬 진짜 원인 (JDBC가 문제였다)

내부적으로 무슨 일이 벌어졌냐면요:

* Spark는 UUID를 `StringType()`으로 읽음
* JDBC는 이를 Postgres에 `VARCHAR`로 전달
* Postgres는 이렇게 말함:

  > "prepared statement에서는 VARCHAR를 UUID로 자동 변환 안 해줌."

일반 SQL에서는 알아서 캐스팅을 해주기도 합니다.
하지만 JDBC prepared statement에서는 절대 자동 변환 안 해줍니다.

심지어 Spark는 `UUIDType()` 같은 것도 지원 안 함 — 그냥 `StringType()`뿐.


## 🚧 내가 시도한 삽질들 

온갖 꼼수를 시도해봤습니다:

* **Spark에서 다시 캐스팅?** → 아무 소용 없음
* **JDBC 옵션 `stringtype=unspecified`?** → 신뢰성 없음
* **Spark SQL에서 억지로 캐스팅 로직 작성?** → 오늘은 되는데 내일 깨짐

깔끔하게 해결되지 않았습니다.


## 🔨 진짜 해결책 (단순하고 효과적)

결국 현실을 받아들였습니다:

> **Postgres에서 UUID를 `TEXT`로 저장.**

```sql
ALTER TABLE fact_transaction ALTER COLUMN transaction_id TYPE TEXT USING transaction_id::TEXT;
```

Fact 테이블과 Dimension 테이블 모두 UUID를 텍스트로 저장하도록 변경.

이유는 간단:

* Kafka는 UUID를 문자열로 생성 ✅
* Spark는 `StringType()`으로 읽음 ✅
* JDBC는 `VARCHAR`로 전달 ✅
* Postgres는 `TEXT`로 잘 받아줌 ✅

이제 모든 게 *그냥 잘 됨*.


## 🧠 업계의 비밀: TEXT도 충분히 괜찮다

사실 많은 사람들이 알려주지 않는 현실:

* Snowflake, BigQuery, Redshift — UUID 타입 없음
* BI 툴? 문자열을 더 선호함
* 디버깅? TEXT가 훨씬 편함
* 스키마 변경? 훨씬 부드러움

OLTP급 트랜잭션 시스템이 아닌 이상, 데이터 파이프라인에서는 UUID를 TEXT로 다루는 게 훨씬 **실용적**입니다.


## 🔄 최종 아키텍처

* Kafka → Spark Streaming → Postgres
* 캐스팅 꼼수 필요 없음
* Docker Compose로 전체 컨테이너 관리
* Dimension 테이블 자동 시딩
* 그냥 `docker compose up` 치면 끝 — 여유롭게 커피 한 잔


## 🚀 교훈 정리

* UUID + Spark + JDBC + Postgres = 지뢰밭 ⚠️
* 분석 파이프라인에선 TEXT > UUID (대부분의 경우)
* 전체 스택에서 타입 시스템을 최대한 단순하게 유지
* 이론적 완벽함보다 **운영의 부드러움**을 최우선으로

결국:
**진짜 데이터 엔지니어링 = 이런 벽에 부딪히는 과정.**
**이걸 넘기면 한 단계 레벨업입니다.**


