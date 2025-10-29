---
title: "Metabase를 Supabase에 안전하게 연결하기 – Read-Only Role, RLS Policy"
date: 2025-10-29
tags: ["supabase", "metabase", "postgres", "rls", "data-visualization", "data-engineering", "security", "dashboard"]
categories: ["Data Engineering"]
---

## 🧭 상황

**Supabase**에 저장된 데이터를 시각화하기 위해 **Metabase 대시보드**를 셋업

## 🔒 Metabase 연결 시 보안 옵션

### **옵션 1. EC2 보안 그룹에서 IP 화이트리스트 설정**

* **집 또는 사무실 IP만 인바운드 접근 허용**
* 효과: 오직 본인만 Metabase 로그인 페이지에 접근 가능
* 단점: 이동 중이거나 다른 네트워크를 사용할 때마다 규칙을 갱신해야 함

### **옵션 2. 리버스 프록시 + 기본 인증 (Basic Auth) 적용**

* 같은 EC2 인스턴스에 **Nginx 리버스 프록시**를 실행
* Nginx에서 다음을 처리:

  * HTTPS (TLS 인증서)
  * Metabase 진입 전 추가 비밀번호 게이트
* 외부에는 **443(HTTPS)** 포트만 노출하고, **3000 포트는 비공개**
* 소규모 팀이 보안을 신경 쓰며 운영할 때 흔히 사용하는 방식

### **옵션 3. 완전 비공개 접근 (내가 선택한 방법)**

Metabase의 3000 포트를 **외부에 전혀 공개하지 않는다.**
대신 **SSH 로컬 포트 포워딩**을 사용한다:

```bash
ssh -i your-key.pem \
    -L 3000:localhost:3000 \
    your-ec2@ip-address
```

이 방식은 SSH 터널을 통해 Metabase에 접속하므로 외부 노출이 없다.

* HTTPS, Nginx, 방화벽 규칙이 필요 없음
* **1인 개발 환경에서 가장 단순하고 안전한 방법**
* 개인용 “founder dashboard”로 개발 중일 때 이상적

팀원이나 투자자와 공유할 시점이 오면,
그때 Nginx + HTTPS를 추가하여 멀티 유저 환경으로 확장가능. 

## 🧩 Supabase 측 설정

### **Metabase 전용 Read-Only 사용자 생성**

Supabase(Postgres)에서는 **`service_role`을 BI 툴에 절대 사용하지 말아야한다.**
대신 데이터를 **조회만 할 수 있는 최소 권한의 사용자(role)** 를 만든다.

이렇게 하면 Metabase가 해킹되더라도 공격자는 데이터를 조회만 할 수 있고,
INSERT / UPDATE / DELETE는 불가능.

```sql
create role metabase_reader login password 'STRONG_PASSWORD';
grant usage on schema public to metabase_reader;
grant select on all tables in schema public to metabase_reader;
alter default privileges in schema public
grant select on tables to metabase_reader;
```

💡 **최소 권한의 원칙 (Principle of Least Privilege)**
Metabase는 오직 SELECT 쿼리 실행 및 테이블 메타데이터 조회만 가능하게 설정.

---

## 🧩 Row-Level Security (RLS) 처리

`SELECT` 권한이 있더라도, RLS 정책이 허용하지 않으면 아무 데이터도 볼 수 없음.

Metabase가 모든 행을 읽을 수 있도록 하려면 정책을 추가한다:

```sql
-- Metabase 읽기 전용 정책 생성
create policy metabase_can_read_all
on public.table
for select
to metabase_reader
using (true);
```

Metabase에서 접근해야 하는 모든 테이블에 대해 이 정책을 추가하거나,
다음 DO 블록으로 일괄 적용.

```sql
do $$
declare r record;
begin
  for r in
    select table_schema, table_name
    from information_schema.tables
    where table_schema in ('public', 'schema2', 'schema3')
      and table_type = 'BASE TABLE'
  loop
    execute format('alter table %I.%I enable row level security;', r.table_schema, r.table_name);
    begin
      execute format('create policy metabase_can_read_all on %I.%I for select to metabase_reader using (true);',
                     r.table_schema, r.table_name);
    exception when duplicate_object then null;
    end;
  end loop;
end$$;
```

이 스크립트는 다음을 수행:

* 대상 스키마의 모든 테이블에 RLS를 활성화
* `metabase_reader`용 전체 읽기 정책 생성
* 이미 동일한 정책이 존재하면 자동으로 건너뜀


## ✅ 마무리

BI 연결의 보안은 **사후 처리 대상이 아니라, 설계의 출발점**.

* **읽기 전용 Postgres 역할 (Read-Only Role)**
* **제한된 네트워크 경로 (SSH 터널)**

간단한 안전 셋업

* 공개 포트 없음
* 민감한 인증 정보 없음
* 우발적인 데이터 유출 없음

