---
title: "Building a Scalable Kubernetes Application on DigitalOcean"
description: "A hands-on Flask demonstration project that shows real-time pod and node assignments during scaling events."
date: 2025-08-30 08:00:00 -0700
categories: [Containerization, Kubernetes Orchestration]
tags: [kubernetes, autoscaling, digitalocean, flask, hpa, cluster-autoscaler, doks, load-balancing]
---

Kubernetes autoscaling remains one of the most powerful yet underutilized features in container orchestration. This demonstration project shows how to implement both horizontal pod autoscaling and cluster node autoscaling on DigitalOcean Kubernetes (DOKS), providing clear visibility into how requests distribute across pods and nodes in real-time.

## The Challenge of Scaling Kubernetes Applications
Setting up autoscaling in Kubernetes involves multiple moving parts. Developers need to configure horizontal pod autoscalers, set appropriate resource limits, and ensure cluster-level scaling works correctly. Without proper visibility into these systems, it becomes difficult to verify that autoscaling actually works as intended or to demonstrate load distribution to stakeholders.
Many teams deploy applications to Kubernetes without fully understanding how their pods scale or how traffic distributes across nodes. This lack of visibility can lead to inefficient resource usage, unexpected costs, or poor application performance under load.

## A Practical Solution for Demonstrating Kubernetes Scalability
This Flask-based demonstration application solves the visibility problem by displaying exactly which pod and node handled each request. The application provides both a web interface and REST API endpoint that return real-time information about the serving pod's identity and its host node.
The project deploys to a DOKS cluster with autoscaling configured at both the pod and node levels. When load increases, the Horizontal Pod Autoscaler (HPA) creates additional pods based on CPU utilization. If the cluster reaches capacity, the Cluster Autoscaler provisions new nodes automatically.

## Key Features and Capabilities
The application exposes critical runtime information that helps developers understand Kubernetes scaling behavior. Each request displays the specific pod name and node assignment, making load balancing immediately visible. Refreshing the page shows different pods responding, demonstrating the LoadBalancer's traffic distribution.
The HPA configuration scales pods from a minimum of 3 replicas to a maximum of 10 based on 70% CPU utilization. The cluster autoscaler adds or removes nodes as needed, with the worker pool scaling between 1 and 3 nodes. This dual-layer scaling ensures the application can handle varying loads efficiently.
A REST API endpoint at `/api/info` provides the same pod and node information in JSON format, enabling programmatic monitoring and integration with other tools. This makes the application useful for both manual testing and automated verification of scaling behavior.

## Technical Implementation Details
The application uses a simple Flask server that reads Kubernetes environment variables and metadata to identify the current pod and node. The Docker container runs with specific resource requests (100m CPU, 128Mi memory) and limits (500m CPU, 256Mi memory) that enable predictable scaling behavior.
Deployment involves creating a DOKS cluster with autoscaling enabled:

```bash
doctl kubernetes cluster create scalable-demo-cluster \
--tag do-scalable \
--auto-upgrade=true \
--node-pool "name=worker-pool;count=2;auto-scale=true;min-nodes=1;max-nodes=3"
```

The Kubernetes manifests define a Deployment, Service, and HorizontalPodAutoscaler. The Service uses type LoadBalancer to obtain an external IP address from DigitalOcean, making the application publicly accessible.
Testing autoscaling requires generating sustained load. The project includes instructions for using DigitalOcean's load generator or creating a simple busybox pod that continuously requests the application. Under load, the HPA creates new pods within 5 minutes of sustained high CPU usage. If cluster capacity is exhausted, the Cluster Autoscaler provisions additional nodes after another 5 minutes.
Monitoring commands help track the scaling process:

```bash
# Watch HPA decisions
kubectl get hpa -w
# Monitor pod creation across nodes
kubectl get pods -o wide -w
# Check resource utilization
kubectl top pods
```

The application integrates with DigitalOcean Container Registry for image storage, though any container registry works with minor configuration changes. The deployment process follows standard Kubernetes patterns, making it easy to adapt for other cloud providers or on-premises clusters.

This demonstration project provides hands-on experience with Kubernetes autoscaling on DigitalOcean. By making pod and node assignments visible, it helps developers understand how Kubernetes distributes load and scales resources. The combination of horizontal pod autoscaling and cluster autoscaling showcases a production-ready scaling strategy that responds to actual demand.
The complete source code and deployment instructions are available on GitHub. Deploy it to your own DOKS cluster to explore Kubernetes scaling behavior or use it as a foundation for building more complex autoscaling demonstrations.
