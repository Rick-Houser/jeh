---
title: SRE Basics - Monitorable Flask App in 5 Steps (Part 1)
description: Turn a Flask app into a monitorable system with tests, logs, Docker, and Prometheus for SRE reliability.
date: 2025-03-22 08:00:00 +0800
categories: [SRE, Observability]
tags: [observability, monitoring, containerization, Prometheus, logging, DevOps, Flask]
---

![](/assets/img/posts/20250322/prometheus_bkg.webp){: width="100%" height="auto" }
_Hero image showing prometheus graph view_

Welcome to the start of our SRE foundations series! Ever wondered how to take a simple web app and turn it into something an SRE would proudly monitor? In this post, I’m kicking off an 8-part series where we’ll start by building a basic observability platform and we’ll evolve that into a fully-featured, scalable, and secure microservices architecture with CI/CD, auto-scaling, advanced observability, and more. Today, we’re covering part 1: setting up a basic Flask app with all the SRE goodies to make it observable–think Prometheus, Grafana, and more. No need to be a Python wizard—this is about the process, not the syntax. 

Let’s dive into the five actions I took to get this app ready for the big leagues, with a focus on SRE skills you can apply to your own projects.

## Step 1: Spin Up a Simple Web App
First things first, I needed something to monitor. I created a Flask app—a lightweight to-do list API with two endpoints: GET /tasks to list tasks and POST /tasks to add one. It’s basic, but that’s the point—start small so we can focus on observability.
- `Takeaway:` Keep your app lean at the start. Complexity comes later; reliability starts now.
- `Action:` Pick a framework (I chose Flask), write a couple of endpoints, and run it locally. Make sure it responds to requests—mine returned an empty list ([]) out of the gate.

```bash
$ curl http://localhost:5000/tasks
[]
```

## Step 2: Add Unit Tests (Yes, SREs Care About This!)
Before focusing on monitoring, I added unit tests. Why? Because SRE isn’t just about watching systems crash—it’s about preventing chaos with solid foundations. I used Python’s unittest to check my endpoints worked as expected—GET returns 200, POST succeeds or fails gracefully.
- `Takeaway:` Tests catch bugs before they hit production, reducing error rates (hint: RED method incoming later). They’re your first line of defense.
- `Action:` Write basic tests for your app’s core functionality. Don’t sweat the details—just ensure it doesn’t blow up. I hit some import snags (like NameError and AttributeError), but sorting those out taught me to keep my module structure tight.


```python
import unittest
from app.app import app

class TestFlaskApp(unittest.TestCase):
    def setUp(self):
        app.config['TESTING'] = True
        self.client = app.test_client()
        # Reset tasks list before each test
        tasks.clear()

    def test_get_tasks_empty(self):
        response = self.client.get('/tasks')
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.json, [])

    def test_add_task_success(self):
        response = self.client.post('/tasks', json={"task": "Buy milk"})
        self.assertEqual(response.status_code, 201)
        self.assertEqual(response.json, {"message": "Task added"})
        # Verify task was added
        response = self.client.get('/tasks')
        self.assertEqual(response.json, ["Buy milk"])

    def test_add_task_no_data(self):
        response = self.client.post('/tasks', json={})
        self.assertEqual(response.status_code, 400)
        self.assertEqual(response.json, {"error": "No task provided"})

if __name__ == '__main__':
    unittest.main()
```

_These basic tests will evolve into a more comprehensive test suite that drives our CI/CD pipeline in future posts._

## Step 3: Log Like You Mean It
Next, I added structured logging with python-json-logger. Instead of plain text like “Hey, someone hit the endpoint,” I got JSON logs—think {"levelname": "INFO", "message": "GET /tasks called"}. Why JSON? Because tools like Loki (spoiler for a later phase) can parse it, making debugging a breeze.
- `Takeaway:` Structured logs are your observability superpower—readable by humans and machines. They’re the first step to understanding what’s happening under the hood.
- `Action:` Add a logging library, sprinkle logger.info() and logger.warning() calls in your code, and test they fire. I updated my tests to capture logs, ensuring I didn’t miss a beat.

![Desktop View](/assets/img/posts/20250322/json_logs.png){: width="972" height="589" }
_Image showing JSON logs from a Flask App_

_Although we’re starting with basic logging now, we’ll expand on this foundation with advanced log aggregation later._

## Step 4: Containerize It
Time to get modern—I wrapped the app in a Docker container. This locks in dependencies (Flask, logging libs) and ensures it runs the same everywhere. I wrote a Dockerfile, added a .dockerignore to keep junk out, and tested it with a shell script to confirm the API still worked.
- `Takeaway:` Containers are your ticket to consistency. No more “works on my machine” excuses—Docker makes it repeatable, which is gold for reliability.
- `Action:` Create a Dockerfile, build your image (docker build -t my-app .), and run it (docker run -p 5000:5000 my-app). Hit it with a request to see it in action. Bonus: script a test to automate the check.

> Don’t forget to use a `.dockerignore` file—excluding unnecessary files prevents bloated images and ensures consistent builds across environments.
{: .prompt-warning }

```bash
$ docker build -t my-app .
$ docker run -p 5000:5000 my-app
```

_This containerization approach sets us up for multi-instance deployment with load balancing and eventually Kubernetes orchestration._

## Step 5: Measure It with Prometheus
Finally, I added Prometheus metrics to track what and how fast. Using prometheus-client, I threw in a counter for request totals and a summary for latency. Then, I spun up Prometheus in Docker Compose to scrape those metrics from my app on port 8000. Now I’ve got numbers to watch—request counts ticking up, latency in seconds—ready for dashboards and alerts.
- `Takeaway:` Metrics are the pulse of your system. Start with basics like rate and duration (RED method vibes), and you’re on the path to proactive monitoring.
- `Action:` Add a metrics library, expose an endpoint (I used 8000), and set up Prometheus with Docker Compose. Check http://localhost:9090 to see your metrics live—it’s a great way to confirm everything’s working.

> The RED method (Rate, Errors, Duration) helps SREs monitor system health—focus on these metrics to catch issues before they escalate.
{: .prompt-info }

![Desktop View](/assets/img/posts/20250322/prometheus_01.png){: width="972" height="589" }

_These basic metrics give us visibility now. We’ll expand upon these later with distributed tracing and enhanced dashboards._

# Wrapping Up

## Why This Matters for SRE
This isn’t just a Flask app anymore—it’s a monitorable system. We’ve laid the groundwork with:
- `Testing:` Catching issues early.
- `Logging:` Capturing events for debugging.
- `Containerization:` Ensuring consistency.
- `Metrics:` Quantifying performance.
These are SRE skills in action—building reliability into the bones of your app. Next up, I’ll add Grafana to visualize this data (think USE and RED methods) and Loki for log aggregation, but for now, you’ve got a solid base to replicate.

We’re just scratching the surface of SRE practices here. As we progress through the series, we’ll address other critical aspects like CI/CD pipelines, auto-scaling, incident response, infrastructure as code, and security. This foundation of testability, observability, and containerization is what makes everything else possible.

## Prerequisites for Following Along
If you’re planning to follow this series through until the end, you’ll need:

- Basic knowledge of Python and Docker
- A GitHub account for CI/CD
- An AWS/GCP/Azure account (or equivalent) for cloud deployment
- Willingness to learn Kubernetes basics

Don’t worry if you’re missing some of these–I’ll explain each step as we go.

## Try It Yourself
Want to follow along? Here’s the quick rundown:
1. Build a simple web app (any language works).
2. Add unit tests to keep it honest.
3. Plug in structured logging—JSON’s your friend.
4. Dockerize it for portability.
5. Slap on Prometheus metrics and watch it hum.

> Ensure Docker Compose is installed before running the commands—it’s required to spin up the services.
{: .prompt-tip }

Want to test my setup? Clone my [**repo**](https://github.com/Rick-Houser/system-prism) to try it yourself—I’ve tagged the full setup at **v1.0** for you to explore. Want to chat about it? Drop me an email (link in the sidebar). To run it, grab the repo with ```bash git clone [repo_link] -b v1.0```, then follow these steps:
1. ```docker-compose up``` — spins up the services.
2. ```bash curl http://localhost:5000/tasks``` — tests the app’s endpoint.

## Best Practices Recap
- `Start Small:` A tiny app lets you focus on observability, not feature creep.
- `Test Early:` Catch errors before they’re incidents.
- `Log Smart:` Structured logs pay off down the line.
- `Containerize:` Consistency is king.
- `Measure Everything:` Metrics are your eyes into the system.

## Up Next
In part 2, we’re taking our observability game up a notch by adding Grafana to visualize all those Prometheus metrics we set up. Expect to see slick dashboards showing request rates, latencies, and more—turning raw data into something you can actually see and act on. We’ll hook Grafana into our Docker setup, making it easy to spot trends and troubleshoot issues. Later, in part 3, we’ll add alerting and cloud deployment, and by part 4, we’ll dive deeper with tools like Loki for logs. Stay tuned as we continue to build reliability in the next post!