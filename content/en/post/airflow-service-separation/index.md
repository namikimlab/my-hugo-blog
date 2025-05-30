+++
title = "🔧 Why Do We Split Airflow into init, scheduler, and webserver?"
date = 2025-05-30T12:00:00+09:00
tags = ["airflow", "data-engineering", "orchestration", "ETL", "pipeline","workflow"]
categories = ["Data Engineering"]
draft = false
+++

If you start working with Airflow a bit more seriously, you'll quickly notice that it's usually split into multiple services:

- `airflow-init`  
- `airflow-scheduler`  
- `airflow-webserver`

At first, you may wonder:  
**"Why do we need to split them up like this?"**

Well — this is actually the **standard production architecture**.  
Let’s break it down in simple, practical terms.


## 1️⃣ airflow-init — Preparation Step

Also sometimes called `airflow-db-migrate` or `airflow-bootstrap`.  
This runs **only once** when you initialize Airflow.

### What it does:
- Runs `airflow db upgrade` → keeps the DB schema up-to-date
- Creates the initial admin user

**Important:**  
This is not a long-running service. Once it completes its job, it exits (`exit code 0`).

### Why does it need to run separately?
- Webserver and scheduler won’t even start if the DB isn’t ready.
- Database initialization is always the very first step.

👉 **Simply put:**  
"The prep job that runs before Airflow can fully start."


## 2️⃣ airflow-webserver — The UI

This is the HTTP server that provides Airflow's Web UI (built on Gunicorn).

### What it does:
- Web UI
- REST API
- Role-based access control (RBAC)
- Trigger management
- Log viewer, and basically everything users interact with
- Default port: 8080

👉 **In short:**  
"The screen you see when you open Airflow in your browser."


## 3️⃣ airflow-scheduler — The Engine

This is the heart of Airflow.  
It scans DAGs, schedules tasks, and handles all execution orchestration.

### What it does:
- Periodically scans DAGs
- Manages task states
- Resolves dependencies
- Queues tasks for execution
- Delegates execution requests to the executor

👉 **In short:**  
"The manager that decides what to run and when."


## 🔥 Key Architecture Summary

- `webserver` and `scheduler` are fully decoupled.
- `webserver` only handles UI — it doesn’t execute tasks.
- `scheduler` queues tasks, and **executor** handles actual task execution.
  (In your current setup using `LocalExecutor`, the scheduler directly executes tasks too.)


## 🔧 Why Split Them?

| Reason | Explanation |
| ------ | ----------- |
| Stability | One component can fail without taking down the others |
| Scalability | Scale webserver and scheduler independently |
| Industry Standard | Helm charts and real-world deployments follow this architecture |

---

## 📌 Extra Note

- Starting from Airflow 2.x, this multi-service architecture is the default.
- Practicing this split using `docker-compose` is the most realistic way to learn.

👉 **Bottom line:**  
You're following exactly the right structure.  
An interviewer who sees this will immediately know:  
**"Ah, this person understands production-level Airflow architecture."**

---

## 🔥 TL;DR

- `airflow-init` → preparation  
- `airflow-scheduler` → execution manager  
- `airflow-webserver` → UI


## Bonus: Understanding Executor Types

In Airflow, the `executor` is a **logical component** — not always a separate process.

- The `scheduler` decides **what to run and when**.
- The `executor` is responsible for **how to run it**.

Depending on which executor you use, task execution works differently:

| Executor | How it runs tasks | Separate Process? |
| -------- | ------------------ | ------------------ |
| **LocalExecutor** | Runs tasks directly inside the scheduler process | ❌ No separate process |
| **SequentialExecutor** | Runs one task at a time (for testing) | ❌ |
| **CeleryExecutor** | Uses separate worker processes | ✅ Requires worker processes |
| **KubernetesExecutor** | Spawns Kubernetes pods for each task | ✅ Runs as pods |

