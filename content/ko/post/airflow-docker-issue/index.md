+++
title = "🔧 Airflow Docker 스타트 문제 해결"
date = 2025-05-30T12:00:00+09:00
tags = ["airflow", "docker", "docker-compose", "postgres", "debugging", "data-engineering"]
categories = ["Data Engineering"]
draft = false
+++

Airflow를 Docker로 운영하면서 자주 마주치는 이슈를 정리 

---

## ❗ 문제 1 — .env 파일이 Airflow 컨테이너 안에서 안 보인다

### 🔍 증상 요약

- `.env` 파일은 프로젝트 루트에 존재.
- 하지만 Airflow 컨테이너 안에서는 `load_dotenv()`가 읽지 못함.
- 이유: 
  - Docker는 `.env` 파일을 환경변수로 변환해 넘기긴 하지만,
  - 파일 자체를 컨테이너 내부로 복사하거나 mount하지 않는다.
  - 결국 `load_dotenv()`가 찾을 파일이 없음.

### ✅ 해결법

#### 1️⃣ docker-compose.yml에 volume mount 추가

```yaml
services:
  airflow:
    ...
    volumes:
      - ./dags:/opt/airflow/dags
      - ./.env:/opt/airflow/dags/.env   # ✅ 이 줄 추가
```

이렇게 하면 .env 파일이 Airflow 컨테이너 안의 /opt/airflow/dags/.env 경로로 복사된다.

#### 2️⃣ DAG 코드에서 정확한 경로로 load_dotenv()  
위치를 써줘야한다
```python
from dotenv import load_dotenv
load_dotenv(dotenv_path="/opt/airflow/dags/.env")
```
이제 DAG 스크립트가 컨테이너 내부의 .env 파일을 정상적으로 읽어온다.

#### 3️⃣ 컨테이너 재시작 필요
```bash
docker compose down
docker compose up --build 
```

## ❗ 문제 2 — AWS Credentials 연동
옵션 1: .env에 자격증명 추가  
.env 파일에:
```env
AWS_ACCESS_KEY_ID=your-access-key-id
AWS_SECRET_ACCESS_KEY=your-secret-access-key
```

boto3는 환경변수에서 자동으로 자격증명을 가져온다:
```python
import boto3
import os
from dotenv import load_dotenv

load_dotenv(dotenv_path="/opt/airflow/dags/.env")
s3 = boto3.client("s3")  # 키를 따로 넘길 필요 없음
```
옵션 2: IAM Role이나 다른 방식도 가능 (여기선 생략)

## ❗ 문제 3 — boto3 디렉토리 문제
boto3의 download_file() 사용할 때는 로컬 디렉토리가 먼저 존재해야 함.
```python
os.makedirs(os.path.dirname(LOCAL_FILE_PATH), exist_ok=True)
```
이렇게 미리 디렉토리를 만들어주면 디렉토리 없음 오류 방지 가능.

## ❗ 문제 4 — Airflow Metadata Database와 Local Files 구분
Airflow의 metadata database (Postgres)는 정상적으로 컨테이너 안에서 잘 돌아감 → 이건 docker-compose로 이미 잘 세팅되어 있음.

하지만 Airflow 컨테이너 내부에는 DB 외에도 local file들이 필요.

| 데이터 종류 | 저장 경로 | 설명 |
|------------|-----------|------|
| 로그 | `/opt/airflow/logs` | task 실행 로그 |
| DAG 스크립트 | `/opt/airflow/dags` | DAG 코드 |
| 설정 및 확장 | `/opt/airflow/plugins`, `airflow.cfg`, `webserver_config.py` | 설정 및 확장 기능 |

### 🔬 핵심 문제의 진짜 원인

- metadata DB (Postgres)는 정상적으로 작동 중.
- 하지만 `airflow webserver`가 내부 폴더 (예: `/opt/airflow/logs/`) 생성을 시도할 때 → volume mount가 안 되어 있어 실패.
- 이로 인해 마치 `airflow db init` 이 실패한 것처럼 보이는 착시 현상이 발생.

### 해결법 — logs volume mount 추가
docker-compose.yml 수정:
```yaml
services:
  airflow:
    ...
    volumes:
      - ./dags:/opt/airflow/dags
      - ./logs:/opt/airflow/logs   # ✅ 이 줄 추가
```
이렇게 하면 webserver가 정상적으로 로그 폴더에 접근 가능 → task 실행과 UI 모두 정상 동작.

## 🔥 최종 요약
- metadata DB와 local files는 완전히 별개의 영역이다.
- local files (logs, dags, plugins)는 컨테이너 내부에 volume mount가 필요하다.
- .env 파일도 volume으로 명시적으로 mount하고 load_dotenv()에서 경로를 지정해야 한다.
- logs volume을 mount하지 않으면 webserver가 내부 오류로 중단될 수 있다.