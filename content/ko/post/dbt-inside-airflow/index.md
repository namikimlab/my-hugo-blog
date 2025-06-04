---
title: "🧚 왜 dbt를 Airflow Docker 컨테이너 안에서 실행하는가"
date: 2025-06-04
tags: ["data-engineering", "airflow", "dbt", "docker", "orchestration"]
categories: ["Data Engineering"]
---

데이터 엔지니어링 파이프라인에서 dbt와 Airflow는 종종 함께 사용된다. 여기서 자주 마주하는 설계 결정이 있다: **dbt를 Airflow와 어떻게 함께 실행할 것인가?**

- dbt를 별도의 컨테이너에서 실행하고 API나 CLI 호출로 오케스트레이션할 것인가?
- 아니면 dbt를 Airflow의 Docker 컨테이너 안에서 직접 실행할 것인가?

둘 다 실험해본 결과, dbt를 Airflow 컨테이너 안에서 실행하는 방식을 선호한다.  

이유는 다음과 같다.

## ✅ 하나의 컨테이너 = 하나의 환경

- Airflow DAG가 동일한 컨테이너 안에서 dbt 명령어를 직접 실행한다.
- 이를 통해 다음이 보장된다:
  - 동일한 Python 버전
  - 동일한 dbt 버전
  - 동일한 의존성 버전 (dbt 패키지, 어댑터)
  - 컨테이너 간 네트워킹 이슈 없음

별도의 컨테이너로 분리할 경우 다음을 관리해야 한다:

- 두 개의 Docker 이미지 유지
- dbt 버전 수동 동기화
- 볼륨 마운트, 파일 접근 등 컨테이너 간 파일 시스템 문제 해결


## ✅ 간소화된 의존성 관리

- Airflow가 dbt 환경을 완전히 제어한다.
- 컨테이너 간 API 또는 CLI 호출을 노출할 필요가 없다.
- 패키지 업그레이드 (`dbt`, `dbt-bigquery`, `dbt-postgres` 등)가 자동으로 동기화된다.

CI/CD 파이프라인에서는 Airflow와 dbt가 모두 설치된 단일 Docker 이미지만 빌드하면 된다 — 유지보수가 훨씬 단순하다.


## ✅ 깔끔한 DAG 코드

Airflow 태스크에서 단순히 다음과 같이 실행 가능하다:

```python
BashOperator(
    task_id="run_dbt",
    bash_command="dbt run --project-dir /opt/airflow/dbt"
)
```

## ✅ 쉬운 로컬 개발

- 하나의 `docker-compose.yml`로 모든 서비스 실행.
- dbt와 Airflow 컨테이너 간 복잡한 네트워킹이나 공유 볼륨 설정 불필요.
- "dbt가 Airflow 안에 제대로 설치됐나? 이 볼륨 마운트가 존재하나?" 이런 고민 불필요.
- 그냥 동작한다.

## ✅ 쉬운 디버깅

dbt가 Airflow 안에서 실패할 경우:

- 로그가 Airflow에 완전히 캡처된다.
- dbt 에러 메시지를 Airflow UI에서 바로 확인 가능.
- 컨테이너나 로그 파일을 왔다갔다 할 필요 없다.

## ✅ 일관된 데이터 엔지니어링 철학: 오케스트레이션이 실행을 소유해야 한다

- Airflow가 dbt를 오케스트레이션하므로, dbt의 런타임까지 소유해야 한다.
- Docker 이미지는 작업 단위를 완전히 반영해야 하며, 쪼개진 마이크로 컨테이너보다는 일체형이 낫다.

## 🔧 나의 일반적인 셋업

Airflow Docker 이미지를 확장하여 dbt를 설치한다:
```dockerfile
FROM apache/airflow:2.9.0

USER root

RUN pip install dbt-core dbt-bigquery dbt-postgres dbt-snowflake

USER airflow
```
- DAG와 dbt 프로젝트 모두 Airflow 공유 볼륨 (/opt/airflow/) 안에 존재.
- 개발, 스테이징, 운영 모두 동일한 환경 유지.

## 🚀 언제 별도 컨테이너를 사용할까?

- 매우 큰 dbt 작업을 오토스케일링 워커(예: Kubernetes Executor)로 실행할 때
- 완전 관리형 dbt Cloud를 사용할 때
- 멀티테넌트 dbt 서비스 아키텍처일 때

일반적인 데이터 엔지니어링 파이프라인의 95%에서는 Airflow 컨테이너 안에서 dbt를 실행하는 것이 훨씬 단순하고, 빠르고, 유지보수성이 좋다.

## 🔥 결론

> “dbt를 Airflow 컨테이너 안에서 직접 실행함으로써 환경 일관성을 보장하고, 오케스트레이션을 단순화하며, 실패 지점을 최소화한다. 이로써 DAG는 휴대성이 높고 디버깅이 쉬워진다.”


