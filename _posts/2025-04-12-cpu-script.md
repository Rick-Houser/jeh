---
title: "Monitoring CPU Usage with Python for System Reliability"
description: "A Python script to monitor CPU usage and send email alerts when it exceeds a threshold, ensuring system health."
date: 2025-04-12 08:00:00 -0700
categories: [Automation, Monitoring]
tags: [python, psutil, smtplib, monitoring, alerting, error handling, automation, devops, sre, scripting]
---

Picture a busy application struggling to keep up as users flood in—constant checks can spot trouble before it impacts performance. This post explores my method for tracking CPU usage and issuing alerts when it crosses a set limit, a vital task for keeping systems stable.

## Problem
Teams must keep systems stable by detecting high CPU usage early. This example uses a Python script to measure CPU usage and send an email if it goes beyond a set limit.

## Solution Code
I created a Python script to track CPU usage and issue email notifications:

```python
import psutil
import smtplib
import logging
from email.mime.text import MIMEText
import time

logging.basicConfig(level=logging.INFO)
last_alert_time = 0
cooldown = 300  # 5 minutes

def check_cpu_usage(threshold=80):
    """Check CPU usage and send an alert if it exceeds the threshold."""
    global last_alert_time
    try:
        cpu_usage = psutil.cpu_percent(interval=1)
        if cpu_usage > threshold and time.time() - last_alert_time > cooldown:
            send_alert(cpu_usage)
            last_alert_time = time.time()
    except Exception as e:
        logging.error(f"Error checking CPU usage: {e}")

def send_alert(cpu_usage):
    try:
        msg = MIMEText(f"High CPU usage detected: {cpu_usage}%")
        msg['Subject'] = 'CPU Usage Alert'
        msg['From'] = 'monitor@example.com'
        msg['To'] = 'admin@example.com'
        
        with smtplib.SMTP('smtp.example.com', 587) as server:  # Replace with your SMTP server
            server.starttls()
            server.login('user@example.com', 'password')  # Add your credentials
            server.send_message(msg)
    except Exception as e:
        logging.error(f"Failed to send alert: {e}")

# Example usage
check_cpu_usage()
```

## Why This Matters
Ensuring smooth user experiences in busy applications requires constant system checks. This script offers an easy way to track CPU usage and notify teams of potential issues.

## Problem-Solving Approach
- **Goal**: Track CPU usage and notify if it surpasses 80%.
* **Tools**: `psutil` for system metrics across platforms, `smtplib` for email notifications. I picked `psutil` for its ease of use and `smtplib` for straightforward email delivery.
* **Steps**: Capture CPU usage, check against the limit, and dispatch an email if needed.

## Design Choices
* **psutil**: `cpu_percent(interval=1)` measures usage over one second, offering a good mix of speed and accuracy.
- **Limit**: 80% works as a starting point, adjustable for different setups.
* **Email Notifications**: SMTP keeps things simple; in production, tools like Prometheus could expand monitoring capabilities.

## Performance Considerations
* **Minimal Impact**:`psutil` runs efficiently, keeping resource usage low.
* **Timing and Control**: A one second interval allows quick detection, while a five minute cooldown avoids excessive notifications.

## Error Handling
* **Usage Check Failures**: If `cpu_percent()` encounters issues (e.g., permission errors), it logs the problem, allowing the script to continue.
* **Notification Issues**: Email failures (e.g., SMTP server unavailable) are logged without interrupting the monitoring process, ensuring consistent operation.

## Real-World Fit
This tool helps teams overseeing busy applications maintain performance by catching CPU spikes early. Including additional metrics like memory or disk space would offer a wider view of system status.

## Visualizing the Process
Here’s a Mermaid flowchart of the monitoring process:
![Desktop View](/assets/img/posts/202504012/cpu-script.png){: width="100%" height="auto" }
_cpu monitoring flow chart_

## What I’d Do Differently
For production, I’d:

* Include command line options (e.g., `python script.py --threshold 75`).
* Store email settings in a configuration file (e.g., server, credentials).
These changes would enhance flexibility across setups. Command line options enable quick limit adjustments, while a configuration file streamlines email setup for various environments.