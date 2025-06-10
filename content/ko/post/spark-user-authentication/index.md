---
title: "🛡️ Spark Docker 스트리밍에서 Kerberos 사용자 인증 문제 해결"
date: 2025-06-07
tags: ["spark", "kafka", "docker", "data-engineering", "kerberos", "streaming"]
categories: ["Data Engineering"]
---

Spark, Kafka, Docker를 사용하여 실시간 스트리밍 파이프라인을 구축하던 중, Kerberos를 사용하지도 않았는데 Kerberos 인증과 관련된 **Spark 오류**가 발생했습니다.

```bash
org.apache.hadoop.security.KerberosAuthException: failure to login: javax.security.auth.login.LoginException: java.lang.NullPointerException: invalid null input: name
```

## ❓ 문제가 발생한 원인은?

* 공식 apache/spark:3.5.0 Docker 이미지를 사용하고 있었습니다.
* Docker 내부의 Spark가 Hadoop의 기본 인증 메커니즘을 해결하려고 시도했습니다.
* Hadoop은 다음을 통해 현재 OS 사용자를 검색하려고 했습니다:

```java
UnixPrincipal(name)
```
* Docker 컨테이너 내부에서 앱이 **적절한 사용자명 매핑이 없는** UID/GID로 실행되고 있었습니다.
* 이로 인해 다음과 같은 오류가 발생했습니다:

```bash
invalid null input: name
```
UnixPrincipal()이 null을 받았기 때문입니다.

## 🔎 근본 원인

* Spark는 내부적으로 Hadoop의 UserGroupInformation을 사용합니다.
* 명시적으로 구성된 사용자가 없으면 Hadoop은 시스템 사용자로 폴백합니다.
* 하지만 Docker 컨테이너는 동적으로 할당된 사용자에 대해 항상 유효한 /etc/passwd 항목을 가지고 있지 않습니다.
* 유효한 사용자명이 없음 → Hadoop 크래시 → Kerberos 예외.

> ⚠️ **참고:** 이 문제는 Kerberos를 사용하지 않아도 발생합니다! 예외 이름이 오해를 불러일으키는데, 단순히 Hadoop이 현재 사용자를 찾지 못하는 것입니다.

## 🔧 해결책: Docker 내부에서 유효한 사용자명 설정

Spark나 Hadoop 설정을 패치하는 대신, 다음과 같이 깔끔하게 해결했습니다:

**1️⃣ Dockerfile 내부에서 명시적으로 유효한 사용자 생성**
```dockerfile
FROM apache/spark:3.5.0
USER root
# airflow 사용자 생성 (또는 다른 유효한 사용자)
RUN useradd --create-home --shell /bin/bash airflow
USER airflow
WORKDIR /opt/spark-app
```

**2️⃣ docker-compose.yml에서 HOME 환경 변수 설정**
```yaml
environment:
  - HOME=/home/airflow
  - HADOOP_USER_NAME=airflow
```

**3️⃣** ✅ 이제 Hadoop의 UserGroupInformation이 컨테이너 내부에서 사용자 신원을 성공적으로 해결할 수 있습니다.

## 🔬 이 해결책이 작동하는 이유

* Hadoop은 UID나 GID를 신경 쓰지 않습니다.
* 유효한 사용자명 문자열만 필요합니다.
* Docker 내부에서 명명된 사용자를 생성하면 이 요구사항을 충족합니다.
* 완전한 Kerberos, Kinit 또는 보안 티켓 인프라를 구성할 필요가 없습니다.

## ✅ 주요 포인트

* Spark 내부의 Hadoop은 유효한 OS 사용자에 의존합니다.
* Docker는 때때로 익명 UID를 생성합니다 → 항상 명시적으로 사용자를 생성하세요.
* HOME과 HADOOP_USER_NAME을 일치하도록 설정하세요.
* SPARK_CLASSPATH=* 스타일의 와일드카드 설정을 피하고 명시적인 JAR 마운팅을 선호하세요.
* 체계적으로 디버깅하세요: 로그를 위에서 아래로 읽으세요. 근본 원인은 보통 생각보다 훨씬 앞서 있습니다.

이 작은 Dockerfile 조정으로 몇 시간의 고통스러운 디버깅을 절약할 수 있었습니다.