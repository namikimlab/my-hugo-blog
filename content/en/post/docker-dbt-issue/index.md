+++
title = "🔧 ARM Mac + Docker + dbt: Troubleshooting Startup Issues"
date = 2025-05-30T12:00:00+09:00
tags = ["airflow", "dbt", "docker", "data-engineering", "debugging"]
categories = ["Data Engineering"]
draft = false
+++

While setting up Airflow + dbt projects with Docker, you may run into this common error message and its solutions.

## 🔍 Problem 1: Platform Architecture Mismatch

Error message:
```bash
The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8)
```

- My Mac is running on ARM (Apple Silicon - M1/M2/M3).
- The official dbt Docker image is built for amd64 (x86-based).

As a result, Docker tries to run cross-architecture using QEMU emulation, which sometimes leads to internal Python path issues → surfaces as the `dbt dbt --version` error.  
This is not a simple dbt bug — the root cause is **platform mismatch.**

## Why does this happen?
- dbt-labs does not yet officially provide ARM-native Docker images.
- Most production environments (AWS, GCP, etc.) still run on amd64.
- M1/M2 developers frequently run into these cross-platform issues.

## Solution


### 1️⃣ Declare platform in docker-compose.yml

```yaml
dbt:
  build:
    context: ./dbt
    dockerfile: Dockerfile.dbt
  platform: linux/amd64 <---- ✅ Add this line
    - ./dbt:/usr/app
  working_dir: /usr/app
  environment:
    - DBT_PROFILES_DIR=/usr/app/profiles
  depends_on:
    - postgres

```
👉 Adding `platform: linux/amd64` tells Docker to explicitly use cross-arch emulation.

### 2️⃣ Check your Docker buildx installation
```bash
docker buildx ls
```

### 3️⃣ Clean cache and rebuild 
```bash
docker-compose build --no-cache dbt
docker-compose up dbt
```

## ❌❌❌ Still not working!!

### The true root cause revealed
- docker-compose is not actually building your Dockerfile.dbt.
- Instead, it's pulling ghcr.io/dbt-labs/dbt-postgres:1.7.9 directly.
- This means the CMD defined in your Dockerfile is never applied.
- Which leads to the error:

```bash
Error: No such command 'dbt'
```

## Final solution: Drop build, use image directly
dbt doesn't need a custom build — using the official image directly is cleaner.

The official dbt Docker image (e.g. ghcr.io/dbt-labs/dbt-postgres:1.7.9) already includes:
- dbt-core 
- dbt-postgres adapter
- Python dependencies
- CMD instructions

So, no need to create your own Dockerfile unless you need something extra.

### 🛠 When would you need a custom build?
- Installing additional dbt plugins (e.g., dbt-bigquery, dbt-snowflake, etc.)
- Adding extra Python packages
- Including custom scripts

### ✅ In our current case:
- We're using the official dbt-postgres image.
- No extra packages are needed.
- Simply mount the profiles directory for configuration.

👉 In this situation, pulling and using the official image directly is more stable and much simpler.

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

### ✅ Key changes:
- Removed build:
- Added image:
- Dockerfile.dbt is no longer needed

## 🔥 TL;DR
This is a very common issue when running Docker on ARM Macs:
- Platform mismatch → specify platform in Docker Compose.
- Skip Dockerfile and use official image → eliminate duplication.
- The official dbt images are fully pre-built and ready to go.
- Attempting unnecessary builds often leads to avoidable problems.
- Just pulling and using the official image is a more production-grade practice.
- For Airflow → custom build may still be needed → keep Dockerfile.