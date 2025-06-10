---
title: "🫙 Spark Streaming의 마지막 장애물: 왜 --jars 만으로 Kafka가 안 될까"
date: 2025-06-07
tags: ["spark", "kafka", "streaming", "data-engineering"]
categories: ["Data Engineering"]
---

데이터 엔지니어로서 복잡한 분산 시스템을 구축하면서 마지막 장애물을 넘었을 때 느끼는 성취감은 정말 특별합니다.  

오늘은 **Apache Spark Structured Streaming + Kafka**를 사용할 때 굉장히 답답한 문제를 공유하려고 합니다:  

👉 바로 악명 높은 `Failed to find data source: kafka` 에러입니다.


## 🧨 문제: 모든 게 정상인데 Kafka만 안 된다

상황을 떠올려봅시다:

- Spark 클러스터 정상 구동
- Postgres 연결도 문제 없음
- Kafka에서 이벤트도 잘 발행됨
- 코드에서 `.readStream.format("kafka")` 호출

그런데 갑자기 다음과 같은 에러가 발생:

```bash
pyspark.errors.exceptions.captured.AnalysisException: Failed to find data source: kafka.
````

그래서 모든 걸 다시 확인해 봅니다:

* ✅ JAR 파일 잘 마운트됨
* ✅ classpath도 정상
* ✅ Kafka와 Spark 버전 호환

그런데도 Spark는 계속 불평합니다.
이게 바로 Spark에서 자주 겪는 숨겨진 함정입니다.

## 🧪 근본 원인: 단순 JAR 로딩 vs. 플러그인 등록

문제는 JAR 파일이 없는 게 아닙니다.
Spark가 **플러그인을 내부적으로 어떻게 등록하는가**의 차이입니다.

조금 더 자세히 살펴볼게요:

### JDBC: 쉬운 케이스

JDBC를 사용할 땐:

1️⃣ `df.write.jdbc()` 호출
2️⃣ Spark가 classpath에서 JDBC 드라이버 검색
3️⃣ 끝 — `--jars` 만으로 충분

### Kafka: 특별한 케이스

Kafka (Structured Streaming)는 다릅니다:

1️⃣ `df.readStream.format("kafka")` 호출
2️⃣ Spark가 Kafka를 **DataSourceV2 플러그인**으로 등록 시도
3️⃣ Spark 초기화 과정에서 플러그인 등록 진행
4️⃣ 단순히 classpath에 JAR 추가하는 것만으로는 등록이 완료되지 않음

그래서 **Kafka에서는 `--jars`만으로 안 되는 경우가 많습니다**.


## 🔧 해결책: 실제로 동작하는 두 가지 방법

### ✅ 방법 1: 하이브리드 방식 (프로덕션 추천)

Kafka 커넥터는 `--packages`로, 나머지 드라이버는 로컬 JAR로 관리:

```bash
/opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0 \
  --jars /opt/spark/extra-jars/postgresql-42.6.0.jar,/opt/spark/extra-jars/kafka-clients-3.5.0.jar \
  --conf spark.driver.extraClassPath="/opt/spark/extra-jars/*" \
  --conf spark.executor.extraClassPath="/opt/spark/extra-jars/*" \
  transaction_streaming.py
```

**왜 이게 잘 동작하나?**
`--packages`는 단순히 JAR만 추가하는 게 아니라, **플러그인 등록까지 내부적으로 처리**해 줍니다.


### ✅ 방법 2: 순수 로컬 JAR 방식 (풀 도커 환경에서 선호)

전체 제어권을 유지하고 싶다면:

```bash
/opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --jars /opt/spark/extra-jars/postgresql-42.6.0.jar,/opt/spark/extra-jars/spark-sql-kafka-0-10_2.12-3.5.0.jar \
  --conf spark.sql.streaming.kafka.allowAutoTopicCreation=true \
  --conf spark.driver.extraClassPath="/opt/spark/extra-jars/*" \
  --conf spark.executor.extraClassPath="/opt/spark/extra-jars/*" \
  transaction_streaming.py
```

⚠️ 주의: **Kafka의 모든 종속 JAR (transitive dependencies)** 를 로컬에 직접 준비해야 합니다.



## 🧭 이 차이가 중요한 이유

* 🔍 **디버깅** — 플러그인 등록 방식을 알면 끝없는 삽질을 줄일 수 있음
* 🚢 **배포 선택지** — Docker 환경은 로컬 JAR 선호, 클라우드 네이티브는 `--packages`가 더 간편
* ⏱ **성능** — `--packages`는 실행 시 다운로드 → 초기 부팅 약간 느리지만 버전 호환 보장
* 👥 **팀 협업** — 명확한 문서화가 "내 컴퓨터에선 되는데?" 상황 방지



## 🚀 프로덕션에서 내가 따르는 룰

| 컴포넌트         | 추천 방식           |
| ------------ | --------------- |
| Kafka 커넥터    | `--packages` 사용 |
| DB 드라이버      | 로컬 `--jars` 사용  |
| 내부 커스텀 라이브러리 | 로컬 `--jars` 사용  |
| 팀 환경         | 사용 방식을 반드시 문서화  |



## 🧵 한 단계 더 깊이

이 문제는 **Spark 내부 동작을 이해하는 것이 왜 중요한지**를 잘 보여줍니다.

> classpath에 JAR이 있느냐 vs. data source가 플러그인으로 등록됐느냐 —
> 이 차이가 스트리밍 파이프라인이 잘 도는지, 디버깅 지옥에 빠지는지를 결정합니다.

다음번에 아래 에러를 만나면:

```bash
Failed to find data source: kafka
```

→ JAR이 있는지만 보지 말고,
→ Spark가 **어떻게** 플러그인을 등록하는지 확인하세요.

마지막 장애물은 사실 어려운 게 아니라
그냥 예상 밖일 뿐입니다.
