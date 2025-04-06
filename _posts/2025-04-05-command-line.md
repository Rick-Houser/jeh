---
title: "From Command Line to Observability: The Evolution of System Introspection"
description: "A follow-up to my earlier analysis of the ls *c command on Medium."
date: 2025-04-05 08:00:00 -0800
categories: [Observability, Tools]
tags: [cli-tools, structured-logging, system-call-tracing, iac, kubernetes, prometheus, grafana]
---

A few years back, I [**shared a quick guide on the `ls *c` command**](https://medium.com/@webrickh/what-happens-when-you-type-ls-c-7c72166a9114){:target="_blank"}. That post broke down a simple way to list files, laying the groundwork for understanding system introspection. Now, with infrastructure more complex, we’ve moved beyond basic command-line tools to advanced observability methods, though those early lessons still matter.

## Beyond ls: Modern CLI Tools for System Introspection
While `ls *c` remains useful for quickly finding files with specific patterns, the modern SRE toolkit includes more powerful alternatives that provide richer information about our systems:

```bash
# Modern alternatives to ls
exa --git -l              # Enhanced ls with git status
fd -e c                   # Find files ending in .c more efficiently
bat main.c                # Syntax-highlighted file viewer
```

These modern alternatives provide additional context like git status, better visualization of directory sizes, and more efficient ways to find and analyze files. The evolution of these tools reflects our need to quickly understand increasingly complex systems.

## From Command Line to Structured Logging
The transition from simple terminal output to structured, searchable logs represents a significant
evolution in system introspection:

```bash
# Traditional approach
grep "ERROR" /var/log/application | grep "*c"

# Modern approach with structured logging
journalctl -u application-service -o json | jq 'select(.PRIORITY==3 and .MESSAGE | contains("*c"))'
```

Structured logging allows us to query, filter, and analyze system behavior with much greater precision. Tools like Loki, Elasticsearch, and Splunk extend these capabilities even further, enabling us to
correlate events across distributed systems.

## Tracing System Calls
Understanding what happens when we run a command like ls *c has evolved from simple explanations to sophisticated tracing tools that reveal the entire system call chain:

```bash
# Trace all system calls made by ls *c
strace ls *c

# Trace specific calls and format the output
strace -e trace=execve,openat,stat -f -o trace.log ls *c
```

For containers and Kubernetes environments, tools like Falco extend this capability to monitor system calls at scale, providing security insights and operational awareness:

```yaml
- rule: Pattern Matching Operations
  desc: Detect when pattern matching operations are performed
  condition: process.name = "ls" and proc.args contains "*c"
  output: "Glob pattern matching detected (user=%user.name pattern=%proc.args)"
  priority: INFO
```

## Observability as the New Paradigm
Modern system introspection extends beyond command-line exploration to comprehensive observability. The three pillars of observability – logs, metrics, and traces – give us a complete picture
of system behavior:

```bash
# Collecting Prometheus metrics about file operations
curl -s http://localhost:9090/metrics | grep file_operations

# Querying pattern matching operations with PromQL
rate(shell_glob_expansions_total{pattern="*c"}[5m])
```

Tools like Grafana, Prometheus, and OpenTelemetry allow us to visualize and correlate data across the entire infrastructure, helping us understand not just what files exist (as ls *c would show us), but how
the system is performing and behaving as a whole.

## Infrastructure as Code and Declarative System State
Perhaps the most significant evolution is that we're moving from inspecting systems after they're built to declaring their desired state upfront:

```yaml
# Kubernetes manifest declaring desired system state
apiVersion: apps/v1
kind: Deployment
metadata:
  name: application
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: application:v1
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
      initContainers:
      - name: find-c-files
        image: busybox
        command: ["sh", "-c", "ls *c > /config/c-files.txt"]
```

With tools like Kubernetes, Terraform, and Ansible, we define what we want the system to look like, and then rely on controllers and reconciliation loops to ensure it stays that way. This shifts our focus from inspecting what exists to verifying that reality matches our intentions.

## Conclusion
The simple act of running ls *c to find files matching a pattern has evolved into sophisticated approaches for system introspection and observability. For SRE and DevOps teams, these modern tools provide deep insights into system performance, building on the basics we learned years ago to create reliable and observable infrastructure. By combining these modern tools with a solid grasp of the basics, we can build more reliable, observable, and maintainable systems.
