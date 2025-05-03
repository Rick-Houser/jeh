---
title: "Containerizing Microservices with Docker"
description: "Guide to containerizing Golang, Java, and Python microservices using Docker with Distroless images and multi-stage builds."
date: 2025-05-03 08:00:00 -0700
categories: [Containerization]
tags: [docker, distroless, multi-stage builds, golang, java, python, container security]
---

For an e-commerce application by OpenTelemetry, I containerized the microservices in Golang, Java, and Python to ensure scalability and security. This guide details my approach to the Docker build and run process, using Distroless images and multi-stage builds for efficient, production-ready containers.

## Prerequisites
- Docker installed on an EC2 instance or local machine.
- Source code for microservices (e.g., from a GitHub repository).
- Basic understanding of Golang, Java, and Python build processes.

## Understanding the Docker Build and Run Process
The Docker build and run process for creating and deploying containerized applications consists of three stages:
1. **Dockerfile Creation**: Define the build instructions.
2. **Image Building**: Use `docker build` to create an image.
3. **Container Running**: Launch the container with `docker run`.

For an e-commerce application with multiple microservices, I selected services in popular languages to demonstrate versatility: the product catalog (Golang), frontend (Java), and recommendation (Python) services.

## Local Testing of Microservices
Before containerizing, test each microservice locally to ensure it functions correctly. This step catches errors early, such as missing dependencies or configuration issues, and is standard in DevOps workflows.

- **Golang (Product Catalog)**: Compile and run the service.
  ```bash
  export PRODUCT_CATALOG_PORT=8088
  go build -o product-catalog .
  ./product-catalog
  ```
  Verify the microservice starts, expecting:
  ![Desktop View](/assets/img/posts/20250501/go-binary-execution.png){: width="100%" height="auto" }

- **Java (Frontend)**: Build and run with Gradle.
  ```bash
  export AD_PORT=8080
  ./gradlew installDist
  ./build/install/opentelemetry-demo-ad/bin/Ad
  ```
  Expect to see the Ad service listening on your chosen port.
  ![Desktop View](/assets/img/posts/20250501/java-binary-exec.png){: width="100%" height="auto" }


- **Python (Recommendation Service)**: For local testing, install dependencies and run the service:
  ```bash
  export RECOMMENDATION_PORT=1010
  export PRODUCT_CATALOG_ADDR=localhost:8088
  export OTEL_SERVICE_NAME=recommendation-service
  pip install -r requirements.txt
  python recommendation_server.py
  ```
  Check service functionality (e.g., recommendations API on port 1010).

## Containerizing Microservices: Examples
With the microservices validated locally, apply the Docker build and run process to containerize them, using Distroless images and multi-stage builds for security and efficiency.

### Containerizing a Golang Microservice
The product catalog service, written in Golang, requires a Dockerfile that builds the binary and runs it securely.

```dockerfile
# Build stage
FROM golang:1.24.2 AS builder
WORKDIR /app
COPY go.mod go.sum ./
COPY . .
RUN go mod download && CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o product-catalog

# Final stage
FROM gcr.io/distroless/static-debian11
WORKDIR /app
COPY --from=builder /app/product-catalog /app/product-catalog
COPY ./products/ /app/products/
# Default port for local testing; override with -e PRODUCT_CATALOG_PORT
EXPOSE 8088
ENTRYPOINT ["/app/product-catalog"]
```

- **Multi-stage Build**: The `builder` stage compiles the binary, while the `distroless` stage creates a minimal runtime image, reducing vulnerabilities.
- **Distroless Image**: Uses Google’s `gcr.io/distroless/static-debian11` image, containing only the binary and minimal dependencies.
- **Security**: Disables CGO and runs as non-root by default in Distroless.
- **Product Data**: Includes the ./products/ directory for runtime data access.

Build and run:
```bash
docker build -t product-catalog:v1.0.0 .
docker run \
  -e PRODUCT_CATALOG_PORT=8088 \
  product-catalog:v1.0.0
```

A "Loaded 10 products" message confirms the service is running. If you encounter errors, check `docker logs product-catalog` for errors.

![Desktop View](/assets/img/posts/20250501/docker-go-run.png){: width="100%" height="auto" }
_Docker run output for Go microservice_

### Containerizing a Java Microservice
The Ad service, built with Java and Gradle, requires dependencies.

```dockerfile
# Build stage
FROM gradle:8.14.0-jdk21 AS builder
WORKDIR /app
COPY build.gradle settings.gradle gradlew* ./
COPY gradle ./gradle
COPY src ./src
COPY ./pb/ ./proto
RUN ./gradlew installDist -PprotoSourceDir=./proto --no-daemon

# Final stage
FROM gcr.io/distroless/java21
WORKDIR /app
COPY --from=builder /app/build/install/opentelemetry-demo-ad/lib /app/lib
USER 1000:1000
# Default port for local testing; override with -e AD_PORT
EXPOSE 8080
ENTRYPOINT ["java", "-cp", "/app/lib/*", "oteldemo.AdService"]
```

I used multiple COPY commands to cache Gradle dependencies separately from source. Docker’s COPY requires explicit paths, unlike cp -r, so we kept gradle/ and pb/ separate to avoid restructuring, prioritizing clarity. Direct Java execution (java -cp /app/lib/* oteldemo.AdService) eliminates script overhead, and gcr.io/distroless/java21 minimizes the attack surface for vulnerabilities. The build output below shows cached layers, and the run output confirms the service starts, with SLF4J warnings (addressed in a future logging post).

- **Multi-stage Build**: Separates build (Gradle) and runtime (Distroless Java) environments.
- **Distroless Image**: Ensures a lightweight, secure container.
- **Port Configuration**: Exposes port 8080 for the web application.

Build and run:
```bash
docker build -t ad-service:v1.0.0 .
docker run \
  -e AD_PORT=8080 \
  ad-service:v1.0.0
```

If all went well, you should see a "Java Ad service has started" message. If not, check `docker logs ad-service` for errors.

![Desktop View](/assets/img/posts/20250501/docker-java-run.png){: width="100%" height="auto" }
_Docker run output showing the Java Ad service has started_

### Containerizing a Python Microservice
The recommendation service, written in Python, uses `pip` for dependencies.

```dockerfile
# Build stage
FROM python:3.12-slim-bookworm AS builder
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt && opentelemetry-bootstrap -a install
COPY recommendation_server.py demo_pb2.py demo_pb2_grpc.py logger.py metrics.py ./

# Final stage
FROM python:3.12-slim-bookworm
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /app/recommendation_server.py /app/demo_pb2.py /app/demo_pb2_grpc.py /app/logger.py /app/metrics.py /app/
ENV PATH=/usr/local/bin:$PATH
USER 1000:1000
# Default port for local testing; override with -e RECOMMENDATION_PORT
EXPOSE 1010
ENTRYPOINT ["python3", "recommendation_server.py"]
```

A multi-stage build with `python:3.12-slim-bookworm` excludes `pip` and build artifacts, reducing image size by ~20-30 MB and improving security. Python uses `python:3.12-slim-bookworm` due to compatibility with `grpcio` and OpenTelemetry, unlike Distroless for Go and Java. `EXPOSE 1010` documents the default port, overridden by `-e RECOMMENDATION_PORT`, with `OTEL_SERVICE_NAME` and `PRODUCT_CATALOG_ADDR` passed externally for flexibility. For local testing, run interactively to see logs; use `-d` for detached mode or `-p 1010:1010` to expose the port externally. 

Build and run:
```bash
docker build -t recommendation:v1.0.0 .
docker run \
  -e OTEL_SERVICE_NAME=recommendation-service \
  -e PRODUCT_CATALOG_ADDR=localhost:8088 \
  -e RECOMMENDATION_PORT=1010 \
  recommendation:v1.0.0
```

Confirm the service is running with `docker ps`. If the service fails, check `docker logs recommendation` for errors.

![Desktop View](/assets/img/posts/20250501/docker-python-run.png){: width="100%" height="auto" }
_docker ps output showing the Python recommendation service is running_

## Best Practices
- **Minimal Images**: Use Distroless for Go and Java to reduce size and vulnerabilities. Python may require `python:3.12-slim-bookworm` for compatibility with dependencies like `grpcio`.
- **Version Pinning**: Specify exact versions (e.g., `golang:1.24.2`, `python:3.12-slim-bookworm`) to avoid breaking changes.
- **Non-root Users**: Use non-root users to enhance security. Go Distroless images (e.g., `gcr.io/distroless/static-debian11`) run as non-root by default, while Python (e.g., `gcr.io/distroless/python3-debian12`) and Java (e.g., `gcr.io/distroless/java21`) require explicit `USER nonroot` or `USER 1000:1000`.
- **Separate Repositories**: Avoid monorepo architecture; each microservice should have its own repository for scalability and independence.

**Note**: This post focuses on containerization. Additional practices like `.dockerignore` (to reduce build context) and `HEALTHCHECK` (for Kubernetes health probes) are recommended for production but covered in a future post.

## Takeaways
This containerization process outlines my approach in building secure, observable microservices. In my next post I'll walk through the process of deploying microservices to Kubernetes.