+++
title = "🔧 왜 Airflow는 init, scheduler, webserver를 따로 띄울까?"
date = 2025-05-30T12:00:00+09:00
tags = ["airflow", "data-engineering", "orchestration", "ETL", "pipeline","workflow"]
categories = ["Data Engineering"]
draft = false
+++

Airflow 조금 만지다 보면, 대부분 이렇게 서비스가 나뉘어 돌아가는 걸 보게 됩니다.

- `airflow-init`  
- `airflow-scheduler`  
- `airflow-webserver`

"이걸 굳이 왜 이렇게 쪼개?" 라는 생각이 들 수 있는데요.  
이게 바로 **실전 운영 방식**입니다.  


## 1️⃣ airflow-init — 준비 담당

다른 이름으로 `airflow-db-migrate` 나 `airflow-bootstrap` 이라고 부르기도 합니다.  
**딱 한 번만 실행**되는 프로세스입니다.

### 하는 일:
- `airflow db upgrade` → DB 스키마 최신화 (DB 구조 최신으로 맞춰줌)
- 최초 admin 유저 생성

**중요 포인트:**  
이건 계속 도는 서비스가 아닙니다. 할 일 다 하면 바로 종료 (`exit code 0`).

### 왜 따로 있어야 하냐면?
- DB 준비가 안 된 상태에서는 webserver나 scheduler가 아예 못 올라옴.
- 결국 DB 준비가 Airflow 전체 실행의 첫 단추라서.

👉 **쉽게 말해서:**  
"Airflow가 본격적으로 돌기 전에 준비 작업만 해주는 역할"


## 2️⃣ airflow-webserver — 사람이 보는 화면 담당

Airflow Web UI를 띄우는 HTTP 서버입니다. (gunicorn 기반으로 돌아감)

### 하는 일:
- Web UI
- API 제공
- 권한 관리 (RBAC)
- Trigger 관리
- Log Viewer 등 → 유저가 클릭하고 보는 건 다 여기서 담당
- 기본 포트: 8080

👉 **한마디로:**  
"우리가 브라우저 열고 들어가는 그 화면 보여주는 담당"


## 3️⃣ airflow-scheduler — 엔진 담당

이게 사실 Airflow의 심장입니다.  
DAG들 스캔하고 task 스케줄 잡고, 실행시키고… 다 이놈이 합니다.

### 하는 일:
- DAG 주기적으로 스캔
- Task 상태 관리
- 의존성 확인
- 실행할 task를 queue에 올림
- Worker한테 실행 요청

👉 **한마디로:**  
"Task를 어떻게 언제 실행할지 관리하는 진짜 관리자"


## 🔥 핵심 정리
- `webserver`랑 `scheduler`는 서로 독립적
- `webserver`는 그냥 화면만 담당 → 직접 task 안 돌림
- `scheduler`가 task를 큐에 올리고 → executor가 실행  

## 🔧 그럼 왜 이렇게 굳이 나눌까?

| 이유 | 설명 |
|------|------|
| 안정성 | 한 놈이 죽어도 나머지는 계속 돌아감 |
| 스케일링 | 필요할 때 scheduler나 webserver 따로 확장 가능 |
| 운영표준 | 산업 표준 Airflow Helm chart도 이렇게 분리 |


## 📌 추가로 알아두면 좋은 점

- Airflow 2.x 부터는 원래 이 multi-service 구조가 기본임.
- Docker-compose로 이렇게 나눠서 연습하는 게 실무랑 거의 똑같음.


## 🔥 한 줄로 요약하면:

- `airflow-init` → 준비
- `airflow-scheduler` → 실행
- `airflow-webserver` → 화면

## 보너스: Executor 종류
- 지금은 `LocalExecutor`라 scheduler가 task를 큐에 올리면 → 자기 안에서 직접 실행까지 해버림. 
- 즉, task 실행을 위해 별도 worker 프로세스 띄우지 않음

Airflow에서 `executor`는 **개념적인 컴포넌트**다.  
물리적으로 별도의 서비스나 프로세스가 항상 있는 건 아니다.

- `scheduler`는 task를 보고 "아 이제 이거 실행할 차례다!" 라고 결정
- 실행 자체는 `executor`가 담당
- 어떤 `executor`를 쓰느냐에 따라 실행 방식이 달라진다:

| Executor 종류 | 실행 방식 | 별도 프로세스? |
| ------------- | --------- | --------------- |
| **LocalExecutor** | scheduler 프로세스 안에서 바로 실행 | ❌ (별도 프로세스 없음) |
| **SequentialExecutor** | 거의 테스트용, 한 번에 하나씩 | ❌ |
| **CeleryExecutor** | worker 프로세스들이 따로 있음 | ✅ (worker 프로세스 필요) |
| **KubernetesExecutor** | Kubernetes pod 생성해서 실행 | ✅ (pod로 따로 실행) |
