+++
title = "🔧 ARM Mac + Docker + dbt 시작 트러블 슈팅"
date = 2025-05-30T12:00:00+09:00
tags = ["airflow", "dbt", "docker", "data-engineering", "debugging"]
categories = ["Data Engineering"]
draft = false
+++

Airflow + dbt 프로젝트를 Docker로 세팅하던 중 발생하는 에러 메시지와 해결법.

## 🔍 문제1 : 플랫폼 아키텍처 mismatch

에러 메시지:
```bash
The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8)
```

- 내 Mac은 ARM (Apple Silicon - M1/M2/M3)
- dbt 공식 도커 이미지는 기본적으로 amd64 (x86 기반)

결국 도커가 내부적으로 qemu 에뮬레이션을 돌리다가 Python 경로 꼬임까지 발생 → `dbt dbt --version` 오류로 드러남.  
이건 단순 dbt 오류가 아니라 **플랫폼 mismatch가 근본 원인**입니다.

## 왜 이 문제가 발생할까?

- dbt-labs는 ARM-native 이미지를 아직 공식으로 제공하지 않음.
- 대부분의 실전 운영환경 (AWS, GCP 등)은 amd64 기반.
- M1/M2 개발자들은 cross-platform에서 이걸 자주 경험.

## 해결법

### 1️⃣ docker-compose.yml에 platform 명시

```yaml
dbt:
  build:
    context: ./dbt
    dockerfile: Dockerfile.dbt
  platform: linux/amd64 <---- ✅ 이부분 추가
    - ./dbt:/usr/app
  working_dir: /usr/app
  environment:
    - DBT_PROFILES_DIR=/usr/app/profiles
  depends_on:
    - postgres

```

👉 platform: linux/amd64 추가 → cross-arch를 명확히 선언하고 build 시도 

### 2️⃣ docker buildx 확인 
```bash
docker buildx ls
```

### 3️⃣ 캐시 날리고 빌드 클린하게 재시작
```bash
docker-compose build --no-cache dbt
docker-compose up dbt
```

## ❌❌❌ 그러나 해결되지 않음!!

### 진짜 루트 원인 하나 더!
- docker-compose가 사실상 Dockerfile.dbt를 빌드하지 않고, ghcr.io/dbt-labs/dbt-postgres:1.7.9 이미지를 그대로 pull하고 있었음.
- 그래서 Dockerfile 내부 CMD가 전혀 반영되지 않았음.
- 이게 Error: No such command 'dbt' 문제로 이어짐.

에러 메시지:
```bash
Error: No such command 'dbt'
```

## 최종 해결: 빌드 제거 → image 사용으로 전환
dbt는 별도 빌드 필요 없음 → 공식 image를 바로 쓰는 게 깔끔하다.

dbt-labs에서 제공하는 공식 Docker 이미지 — 예를 들어 ghcr.io/dbt-labs/dbt-postgres:1.7.9 — 이 이미지 안에는 이미 다음이 다 들어있다:
- dbt-core (dbt 본체)
- dbt-postgres adapter
- python 의존성
- 실행 커맨드 (CMD 설정까지 포함됨)

즉, 우리가 직접 Dockerfile을 만들어서 또 뭔가 추가로 빌드할 필요가 없다.

### 🛠 언제 빌드가 필요한가?
- dbt에 플러그인 설치 (예: dbt-bigquery, dbt-snowflake 등 다른 adapter)
- 추가 Python 패키지 설치
- 커스텀 스크립트 포함

이럴 때는 Dockerfile을 만들어서 빌드가 필요하다.

### ✅ 그런데 지금 상황에서는?
- 우리는 dbt-postgres 이미지를 그대로 쓰고 있음.
- 별도 추가 패키지도 없음.
- 그냥 profiles 디렉토리 mount해서 config만 넘겨주면 됨.

👉 이 경우에는 공식 image를 pull 해서 그대로 쓰는 게 가장 안정적이고 깔끔함.
빌드 안 해도 되고, 빌드 꼬일 일도 없음.

```yaml
dbt:
  image: ghcr.io/dbt-labs/dbt-postgres:1.7.9
  platform: linux/amd64
  volumes:
    - ./dbt:/usr/app
  working_dir: /usr/app
  environment:
    - DBT_PROFILES_DIR=/usr/app/profiles
  depends_on:
    - postgres
```

✅ 핵심 변화:
- build: 제거
- image: 선언
- Dockerfile.dbt 삭제 가능

## 🔥 한 줄 요약  
이 문제는 ARM Mac에서 Docker 쓰다보면 마주칠 수 있다. 
- 플랫폼 mismatch → platform 명시
- Dockerfile 안 쓰고 공식 이미지 사용 → 중복 문제 제거
- dbt 공식 이미지는 대부분 "이미 다 세팅된 완제품"이다.
- 필요 없는 빌드를 시도하다가 문제를 만드는 것보다 →
그냥 image만 pull해서 쓰는 게 더 실전적인 운영 방식이다
- airflow → 커스텀 빌드 필요 → Dockerfile 유지