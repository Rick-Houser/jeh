---
title: "Setting Up CI/CD Pipelines with GitHub Actions and Argo CD"
description: "Building automated CI/CD pipelines using GitHub Actions for continuous integration and Argo CD for continuous deployment."
date: 2025-05-04 08:00:00 -0700
categories: [Automation, CI/CD]
tags: [github actions, argo cd, ci/cd, gitops, docker, kubernetes, automation]
---

This article explores building CI/CD pipelines for an e-commerce microservice app with GitHub Actions for continuous integration and Argo CD for continuous deployment, focusing on zero-touch automation and top reliability and security practices.

## Prerequisites
- GitHub repository with microservice code and Kubernetes manifests.
- Docker Hub or AWS ECR for storing images.
- EKS cluster with Argo CD installed.
- AWS CLI and kubectl configured.

## Step 1: Configuring GitHub Actions for CI
Create a GitHub Actions workflow to build, test, and push Docker images:
```yaml
name: CI Pipeline
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    - name: Run static analysis
      run: |
        docker run --rm -v $(pwd):/app ghcr.io/hadolint/hadolint hadolint /app/Dockerfile
    - name: Run unit tests
      run: |
        docker build -t test-image --target test .
        docker run --rm test-image go test ./...
    - name: Build and push image
      run: |
        docker build -t user/product-catalog:${{ github.sha }} .
        docker push user/product-catalog:${{ github.sha }}
```

- **Triggers**: Runs on push or PR to `main`.
- **Static Analysis**: Uses Hadolint to lint the Dockerfile.
- **Unit Tests**: Runs tests within a Docker container.
- **Image Push**: Builds and pushes to Docker Hub, secured with secrets.

## Step 2: Setting Up Argo CD for CD
Argo CD syncs Kubernetes manifests from a Git repository to the EKS cluster. Create an application:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: product-catalog
  namespace: argocd
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  source:
    repoURL: https://github.com/your-repo/ecommerce-manifests
    path: manifests/product-catalog
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Update the deployment manifest with the new image:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-catalog
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: product-catalog
        image: user/product-catalog:${{ github.sha }}
        ports:
        - containerPort: 8088
```

- **GitOps**: Argo CD ensures cluster state matches Git.
- **Automation**: Auto-syncs on manifest changes, pruning unused resources.
- **Security**: Use private repositories with SSH keys.

## Step 3: Integrating CI and CD
When code is pushed to `main`, GitHub Actions builds and pushes the image. A separate workflow updates the manifest repository:
```yaml
name: Update Manifest
on:
  push:
    branches: [ main ]

jobs:
  update-manifest:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        repository: your-repo/ecommerce-manifests
        token: ${{ secrets.GIT_TOKEN }}
    - name: Update image tag
      run: |
        sed -i "s|image: user/product-catalog:.*|image: user/product-catalog:${{ github.sha }}|" manifests/product-catalog/deployment.yaml
        git config user.name "GitHub Action"
        git config user.email "action@github.com"
        git commit -am "Update image to ${{ github.sha }}"
        git push
```

Argo CD detects the manifest change and deploys the new image.

## Best Practices
- **Security**: Store secrets in GitHub Secrets, not in code.
- **Reliability**: Add tests and scans (e.g., Trivy) in CI.
- **Separate Repositories**: Use distinct repositories for code and manifests to avoid monorepo complexity.
- **Monitoring**: Integrate Prometheus to track pipeline failures.

**Production Considerations**: While this setup provides a solid foundation for CI/CD, production environments require additional enhancements. For enterprise-grade deployments, implement vulnerability scanning with tools like Trivy or Snyk, generate Software Bills of Materials (SBOMs) for compliance, and consider progressive delivery strategies such as canary deployments using Argo Rollouts. Implement proper environment promotion workflows from development to production, integrate comprehensive observability with metrics collection, and establish external secrets management beyond GitHub Secrets. Lastly, ensure policy enforcement with tools like OPA or Kyverno to maintain compliance and security guardrails in your deployment process.

## Takeaways
GitHub Actions and Argo CD enable zero-touch CI/CD, automating builds, tests, and deployments. This setup ensures reliable and secure delivery of microservices to Kubernetes. In a future post, I will explore SRE troubleshooting using FlagD for feature flagging and failure simulation.