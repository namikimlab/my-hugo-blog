---
title: "🧹 데이터 클렌징: 왜 스테이징 레이어에서 클렌징을 해야 하는가"
date: 2025-06-04
tags: ["data-engineering", "dbt", "data", "pipeline"]
categories: ["Data Engineering"]
---

실제 데이터 엔지니어링 파이프라인에서 가장 흔한 실수 중 하나는 **데이터 클렌징을 너무 뒤로 미루는 것**입니다.  

업스트림 데이터가 깨끗할수록 다운스트림 모델은 더 단순하고 유지보수하기 쉬워집니다.  

이제 하나씩 살펴보겠습니다.

## ✅ 기본 원칙

가능한 한 **가장 이른 단계에서** 데이터를 클렌징 — 이상적으로는 **스테이징 레이어에서** 처리해야 합니다.


## ✅ 그 이유

### 1️⃣ 책임 분리의 명확성

- **스테이징 모델**의 역할은 다음과 같습니다:
  - 클렌징
  - 표준화 (정규화)
  - 타입 캐스팅

- 만약 더러운 데이터를 marts까지 (예: `dim_customer`, `fct_transaction`) 넘긴다면:

  - 다운스트림 모델이 복잡해짐  
  - 중복 필터 발생  
  - 유지보수가 어려운 취약한 코드 생성


### 2️⃣ Marts 레이어는 비즈니스 로직만 담당해야 한다

- Marts 레이어는 오직 다음에 집중해야 합니다:
  - 조인
  - 집계
  - 차원 관계 구성

- 클렌징, null 처리, 표준화는 **이미 업스트림에서 끝나 있어야** 합니다.
- 이렇게 하면 SQL이 훨씬 깔끔하고 가독성이 높아집니다.

### 3️⃣ 실전 시스템은 이렇게 설계된다

**파이프라인 흐름:**

source → staging (클렌징, 표준화) → marts (모델링, 집계)

## ✅ 요약 표

| 레이어  | 역할                          | 클렌징 적용 여부  |
|--------|--------------------------------|-------------------|
| 스테이징 | 클렌징, 표준화, 타입 캐스팅     | ✅ 반드시 적용 |
| Marts  | 조인, 집계, 비즈니스 로직 | ❌ 클린 데이터만 받아야 함 |


## ✅ 실전 적용 규칙

스테이징 레이어에서는 다음을 적용:

- 더러운 문자열 (`'NaN'`, 빈 문자열, 이상값 등) → `NULL`로 변환  
- **기본키 누락된 행** 삭제 (예: `cardholder_id IS NULL`)  
- 비키 컬럼 (예: `description`)은 `NULL` 허용 — 다운스트림에서 안전하게 처리 가능

## 🔥 면접이나 포트폴리오에서 이렇게 설명하라:

> "스테이징 레이어에서 더러운 문자열을 표준화하여 marts가 항상 클린 데이터를 소비하도록 설계했습니다. 이로 인해 다운스트림 조인과 집계가 단순해지고 시스템 안정성과 유지보수성이 크게 향상되었습니다."



