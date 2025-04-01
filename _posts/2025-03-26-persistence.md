---
title: SRE Basics - Persistence, Logs, and Dashboards with SQLite and Loki (Part 4)
description: Enhance your Flask app’s observability with SQLite persistence, Loki log aggregation, and advanced Grafana dashboards in this SRE-focused guide.
date: 2025-03-26 08:00:00 +0800
categories: [SRE, Observability, Cloud]
tags: [sqlite, loki, promtail, grafana, flask, ec2, docker-compose, monitoring]
---

Let’s continue our SRE journey. In part 3, we added alerting with Alertmanager and deployed our Flask app to AWS EC2, tagging it as `v3.0`. Now, in part 4, we’re enhancing observability with persistent storage, log aggregation, and richer dashboards. These steps make our app more realistic and insightful—core traits of a reliable system. Here’s how I tackled it in four steps, with lessons for your own projects.

## Step 1: Add Persistence with SQLite
I swapped the in-memory task list for SQLite, adding full CRUD functionality—create, read, update, delete. Tasks now persist across restarts, stored in a `tasks.db` file outside the container, mimicking production setups where data lives beyond ephemeral instances.

  > Always mount database files outside the container—internal storage risks data loss on container restarts.
  {: .prompt-warning }

- **Takeaway**: Persistence ensures state survives failures—a must for real-world reliability.
- **Action**: Replace volatile storage with a database; mount it externally for durability.
  
![Desktop View](/assets/img/posts/20250326/sqlite.png){: width="100%" height="auto" }
_CRUD endpoints in app.py using SQLite_

## Step 2: Integrate Loki for Log Aggregation
Next, I added Loki and Promtail to collect and store logs. The app now writes structured JSON logs to a file, which Promtail scrapes and sends to Loki. This gives us a centralized log history, searchable and ready for analysis.

  > Ensure Promtail’s config points to your log file’s exact path to avoid scraping issues.
  {: .prompt-info }

- **Takeaway**: Logs provide context to metrics—crucial for debugging and understanding incidents.
- **Action**: Set up a log aggregator and point your app’s output to it.

![Desktop View](/assets/img/posts/20250326/docker-compose-loki-promtail.png){: width="100%" height="auto" }
_docker-compose file updated with loki and promtail services_

## Step 3: Enhance Grafana Dashboards
With Loki in place, I connected it to Grafana and added a logs panel showing app activity. I also introduced an error rate panel (`rate(error_count_total[5m])`), completing our RED method visualization—rate, errors, duration—all in one view.

- **Takeaway**: Unified metrics and logs in dashboards reveal system behavior at a glance.
- **Action**: Link your log store to Grafana and graph key metrics like error rates.

![Desktop View](/assets/img/posts/20250326/app-logs-error-rate.png){: width="100%" height="auto" }
_Grafana dashboard with logs and error rate panels_

## Step 4: Troubleshoot Persistence Hiccups
SQLite setup hit a snag—`tasks.db` kept appearing as a directory, crashing the app. Pre-creating it as a file and using absolute paths (`/app/tasks.db`) fixed it. This taught me to double-check volume mounts and file assumptions.

  > Use `docker volume ls` to inspect mounts if files aren’t appearing as expected.
  {: .prompt-tip }

- **Takeaway**: Reliability includes solving unexpected quirks—persistence demands precision.
- **Action**: Test storage setups thoroughly; expect the unexpected.
  
```bash
$ curl -X POST -H "Content-Type: application/json" -d '{"task": "Test"}' http://localhost:5000/tasks
```
![Desktop View](/assets/img/posts/20250326/sqlite-log-test.png){: width="100%" height="auto" }
_Grafana logs panel showing the 'Test' task added via curl_

## Why This Matters for SRE
Part 4 builds a system that’s not just monitored but understood. SQLite adds resilience, Loki enriches context, and dashboards tie it together—advancing RED and laying groundwork for USE (Utilization, Saturation, Errors). We’re moving beyond basics to a mature observability stack.

## Try It Yourself
Ready to try this? Here’s how:
1. Add SQLite to your app—implement CRUD and mount the DB file externally.
2. Set up Loki and Promtail—pipe logs through a file to a central store.
3. Enhance Grafana—add logs and error rate panels.
4. Test it—hit your endpoints and watch the dashboard.

Clone my [**repo**](https://github.com/Rick-Houser/system-prism), checkout `v4.0`, and run `docker-compose up -d`.

## Best Practices Recap
- **External Storage**: Keep data outside containers for persistence.
- **Log Structure**: Use JSON for searchable, consistent logs.
- **Dashboard Focus**: Combine metrics and logs for clarity.

## Up Next
Part 5 will explore scaling—load testing, resource monitoring, and maybe a multi-node setup. We’re pushing reliability further—stay tuned!