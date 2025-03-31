---
title: SRE Basics - Advanced Monitoring and Load Testing with Multiple Instances (Part 6)
description: Run two Flask instances with PostgreSQL, monitor USE/RED metrics via Prometheus and Grafana, and load test with Locust to push reliability.
date: 2025-03-29 08:00:00 +0000
categories: [SRE, Observability, Cloud]
tags: [flask, prometheus, grafana, locust, postgresql, docker-compose, monitoring, load-testing, nginx]
---

# SRE Basics - Advanced Monitoring and Load Testing with Multiple Instances (Part 6)

Welcome back! In prior posts, we built a Flask app with basic monitoring, up to `v5.0`. Part 6 advances observability by running two instances with PostgreSQL, tracking USE and RED metrics via Prometheus and Grafana, and testing reliability with Locust. We hit 92% CPU, triggering a `HighCPUUsage` alert, though 50 to 60% failures under load indicate areas to improve. Here’s the five-step process, tailored for SRE skill-building.

## Step 1: Migrate to PostgreSQL
We swapped SQLite for PostgreSQL to support multiple instances, updating `app.py` with `psycopg2` and adding a `db` service in `docker-compose.yml`.

- **Takeaway**: Reliable data layers are foundational for multi-node systems.
- **Action**: Switch to a scalable DB and test concurrency.

![Desktop View](/assets/img/posts/20250329/docker-compose-postgresql.png ){: width="100%" height="auto" }
_PostgreSQL added to docker-compose_

## Step 2: Stabilize with Retry Logic
PostgreSQL startup delays crashed the app (`psycopg2.OperationalError`). We added retry logic in `init_db`, fixed a `NameError` for `update_resources`, and dropped `before_first_request`.

- **Takeaway**: Resilience against failures reduces downtime.
- **Action**: Implement retries for external dependencies.
  
  > Cap retries (e.g., 5 attempts, 2s delay) to avoid infinite loops on persistent failures.
  {: .prompt-tip }

![Desktop View](/assets/img/posts/20250329/db-retry.png ){: width="100%" height="auto" }
_Database retry logic in app.py_

## Step 3: Monitor Multiple Instances
Two Flask instances (`flask-app-1`, `flask-app-2`) ran on port 5000 internally, with Prometheus scraping `cpu_usage_percent` and `request_count_total`. Grafana visualized USE metrics, including utilization and errors.

- **Takeaway**: Aggregated metrics reveal system health across nodes.
- **Action**: Scale instances and monitor with Prometheus and Grafana.
  
  > Label instances in Prometheus (e.g., `instance="flask-app-1"`) for clear Grafana visuals.
  {: .prompt-info }

![Desktop View](/assets/img/posts/20250329/cpu-usage.png ){: width="100%" height="auto" }
_CPU Usage panels as shown in Grafana_

## Step 4: Fix Prometheus Targets
Metrics didn’t reach Grafana because `prometheus.yml` targeted port 8000 while Flask ran on 5000. Logs pinpointed the issue, resolved by updating to `flask-app-1:5000` and `flask-app-2:5000`.

- **Takeaway**: Misaligned configs break observability, so trust your logs.
- **Action**: Verify ports match between app and monitoring.
  
  > Dual ports (5000:5000, 8000:8000) in Docker Compose can mislead, so check `docker logs` to confirm.
  {: .prompt-warning }

```bash $ docker logs flask-app-1
[2025-03-29 02:48:27 +0000] [1] [INFO] Starting gunicorn 23.0.0
[2025-03-29 02:48:27 +0000] [1] [INFO] Listening at: http://0.0.0.0:5000
```

## Step 5: Load Test and Alert
Locust simulated 1000 users at a 100 spawn rate, pushing CPU to 92%. A `HighCPUUsage` alert triggered at over 80% for 5 minutes, but 50 to 60% failures (`RemoteDisconnected`, 500s) revealed strain.

- **Takeaway**: Load tests expose limits, alerts flag issues, and failures provide lessons.
- **Action**: Stress your system and refine alerts.
  
  > High failures (50 to 60%) under load signal scaling limits, so consider tuning Gunicorn workers or adding a load balancer next.
  {: .prompt-warning }

![Desktop View](/assets/img/posts/20250329/HighCPUUsage-alerts.png ){: width="100%" height="auto" }
_HighCPUUsage Alerts firing in Prometheus_
![Desktop View](/assets/img/posts/20250329/locust-failures.png ){: width="100%" height="auto" }
_Locust showing RemoteDisconnected errors_

## Why This Matters for SRE
Part 6 creates an SRE playground with live USE metrics (CPU and memory utilization, errors) and RED metrics (request rates, errors), plus a basic alert. Two instances with Nginx mimic a distributed setup, ideal for chaos testing and recovery drills. The high failure rate sets up our next challenge.

## Try It Yourself
Build your SRE skills:
1. Use PostgreSQL by adding `psycopg2-binary` and a `db` service.
2. Stabilize by retrying DB connections.
3. Scale by running two instances.
4. Monitor by fixing Prometheus ports.
5. Test by loading with Locust and setting an alert.

Clone [insert_repo_link_here], checkout `v6.0`, and run `docker-compose up -d`. Push to Docker Hub: `docker push yourusername/flask-app:v6.0`.

## Best Practices Recap
- **Resilience**: Retries beat crashes.
- **Observability**: Align ports for metrics flow.
- **Testing**: Push limits to learn.

## Up Next
Part 7 will explore Kubernetes and chaos engineering by simulating failures like DB outages and network delays to test recovery and refine alerts, staying true to SRE’s heart.