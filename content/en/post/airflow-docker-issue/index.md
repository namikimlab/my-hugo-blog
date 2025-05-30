+++
title = "🔧 Solving Airflow Docker Startup Issues"
date = 2025-05-30T12:00:00+09:00
tags = ["airflow", "docker", "docker-compose", "postgres", "debugging", "data-engineering"]
categories = ["Data Engineering"]
draft = false
+++

Common issues you will often encounter when running Airflow with Docker.

## ❗ Issue 1 — `.env` file is not visible inside Airflow container

### 🔍 Symptom Summary

- The `.env` file exists at the project root.
- But inside the Airflow container, `load_dotenv()` fails to read it.
- The reason:
  - Docker automatically passes `.env` as environment variables.
  - But Docker does not copy or mount the file itself into the container.
  - Therefore, `load_dotenv()` has no file to read.

### ✅ Solution

#### 1️⃣ Add volume mount for `.env` in `docker-compose.yml`
This way, the `.env` file becomes available inside the container at the correct path.

```yaml
services:
  airflow:
    ...
    volumes:
      - ./dags:/opt/airflow/dags
      - ./.env:/opt/airflow/dags/.env   # ✅ Add this line
```


#### 2️⃣ Specify full path when load_dotenv()  
In your DAG code, specify the full path to `.env` when calling `load_dotenv()`. This ensures the container can read the file properly.

```python
from dotenv import load_dotenv
load_dotenv(dotenv_path="/opt/airflow/dags/.env")
```

#### 3️⃣ restart the container to apply updates.
```bash
docker compose down
docker compose up --build 
```

## ❗ Issue 2 — AWS Credentials
**Option 1:** Add credentials to `.env` file.  

```env
AWS_ACCESS_KEY_ID=your-access-key-id
AWS_SECRET_ACCESS_KEY=your-secret-access-key
```
Boto3 will automatically load AWS credentials from environment variables without needing to explicitly pass keys in your code.

```python
import boto3
import os
from dotenv import load_dotenv

load_dotenv(dotenv_path="/opt/airflow/dags/.env")
s3 = boto3.client("s3")  # no need to pass key 
```
**Option 2:** Using IAM Roles or other methods (not covered here).

## ❗ Issue 3 — boto3 directory issue
When using `boto3.download_file()`, the target local directory must exist beforehand.  
You should create the necessary directories in advance to prevent "directory not found" errors.

```python
os.makedirs(os.path.dirname(LOCAL_FILE_PATH), exist_ok=True)
```


## ❗ Issue 4 — Separation of Airflow Metadata Database and Local Files
The Airflow metadata database (Postgres) works fine inside the container as configured in `docker-compose.yml`.  

However, Airflow also requires local files inside the container beyond just the database.

| Data Type | Path | Description |
|------------|-----------|-------------|
| Logs | `/opt/airflow/logs` | Task execution logs |
| DAG scripts | `/opt/airflow/dags` | DAG code |
| Config & Extensions | `/opt/airflow/plugins`, `airflow.cfg`, `webserver_config.py` | Settings and plugins |

### 🔬 The actual root cause:

- The metadata DB (Postgres) runs properly.
- But when `airflow webserver` tries to create internal folders (e.g. `/opt/airflow/logs/`), it fails because no volume mount exists.
- This failure sometimes appears as if `airflow db init` has failed, which can be misleading.

### Solution — Add logs volume mount
Modify `docker-compose.yml` to mount the logs directory, allowing the webserver to access logs properly and ensure stable task execution and UI operation.

```yaml
services:
  airflow:
    ...
    volumes:
      - ./dags:/opt/airflow/dags
      - ./logs:/opt/airflow/logs   # ✅ Add this line
```


## 🔥 Final Summary
- Metadata DB and local files are completely separate concerns.
- Local files (`logs`, `dags`, `plugins`) must be mounted into the container via volumes.
- The `.env` file also needs to be explicitly mounted and loaded via `load_dotenv()` with the correct path.
- Without mounting the `logs` volume, the webserver may crash due to missing directories.
