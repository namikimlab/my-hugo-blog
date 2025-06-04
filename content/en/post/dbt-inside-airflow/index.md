---
title: "🧚 Why Run dbt Inside Airflow Docker Container"
date: 2025-06-04
tags: ["data-engineering", "airflow", "dbt", "docker", "orchestration"]
categories: ["Data Engineering"]
---

# Why I Run dbt Inside Airflow Docker Container

In modern data engineering pipelines, dbt and Airflow often work side by side. One common design decision is **how to run dbt alongside Airflow**:

- Should dbt run in its own container, orchestrated via API or CLI call?
- Or should dbt run directly inside Airflow’s Docker container as part of the DAG?

After experimenting with both, I prefer running dbt *inside* Airflow's Docker container.  

Here’s why.

---

## ✅ One Container = One Environment

- Airflow DAGs directly execute dbt commands inside the same container.
- This guarantees:
  - Same Python version
  - Same dbt version
  - Same dependency versions (dbt packages, adapters)
  - No cross-container networking issues

With separate containers, you often need to:

- Maintain two separate docker images
- Sync dbt versions manually
- Deal with volume mounts or cross-container file access

---

## ✅ Simplified Dependency Management

- Airflow has full control over dbt's environment.
- No need to expose dbt via API or CLI calls across containers.
- Package upgrades (`dbt`, `dbt-bigquery`, `dbt-postgres`, etc.) are synchronized automatically.

In CI/CD pipelines, you just build *one* Docker image with both Airflow and dbt installed — much simpler to maintain.

---

## ✅ Clean DAG Code

Your Airflow tasks can simply run:

```python
BashOperator(
    task_id="run_dbt",
    bash_command="dbt run --project-dir /opt/airflow/dbt"
)
```

No need for custom dbt API operators or cross-container shell hacks.
Simple is stable.

## ✅ Easier Local Development
- One docker-compose.yml runs everything.
- No complex networking or shared volumes between dbt and Airflow containers.
- You don’t need to figure out: "Is dbt installed correctly inside Airflow? Does this volume mount exist?"
- It simply works.

## ✅ Easier Debugging
When dbt fails inside Airflow:
- Logs are fully captured by Airflow.
- You can see dbt error messages directly in Airflow UI.
- No need to jump between separate containers or log files.

## ✅ Consistent Data Engineering Philosophy: Orchestration Owns Execution
- Airflow orchestrates dbt — it should own dbt’s runtime too.
- Docker images should reflect the entire unit of work, not fragmented micro-containers.

## 🔧 My Typical Setup
Airflow Docker image is extended to install dbt:
```dockerfile
FROM apache/airflow:2.9.0

USER root

RUN pip install dbt-core dbt-bigquery dbt-postgres dbt-snowflake

USER airflow
```
- Both DAGs and dbt projects live inside shared Airflow volume (/opt/airflow/).
- One consistent environment across dev, staging, and production.

## 🚀 When Would I Use Separate Containers?
- Extremely large dbt runs that require autoscaling worker pods (e.g. Kubernetes Executor).
- Fully managed dbt Cloud orchestrations.
- Multi-tenant dbt service architecture.

For 95% of typical data engineering pipelines, running dbt inside Airflow’s container is simpler, faster, and much more maintainable.

## 🔥 Conclusion
> “I run dbt directly inside Airflow containers to guarantee full environment consistency, simplify orchestration, and minimize failure points. This makes DAGs highly portable and much easier to debug.”

