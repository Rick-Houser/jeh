---
title: "Automating a simple CI/CD Pipeline with GitHub Actions"
description: "A step-by-step guide to setting up a CI/CD pipeline for a Node.js app using GitHub Actions and Docker."
date: 2025-04-07 08:00:00 -0800
categories: [DevOps, Automation]
tags: [github actions, docker, node js, ci cd, devops, automation, sre]
---

This post outlines my process of setting up a CI/CD pipeline for a Node.js app using GitHub Actions and Docker. Following my [previous Terraform project](#), I applied automation to streamline building and testing, documenting the steps for a simple workflow.

## Setting Up the Prerequisites
I installed Node.js for the app, Docker for containerization, and set up a GitHub repository to host the pipeline.

**Install Node.js**:
1. Go to [nodejs.org](https://nodejs.org) and download the LTS version (e.g., 18.x).
2. Install it on your system (macOS, Windows, or Linux).
3. Verify: `node --version` (should show v18.x.x).

**Install Docker**:
1. Download Docker Desktop from [docker.com/get-started](https://www.docker.com/get-started).
2. Install and start Docker.
3. Verify: `docker --version`.

**GitHub Repository**: I created a repository named `ci cd demo` on GitHub, using the Node.js `.gitignore` template to exclude files like `node_modules`.

> Verify tool versions (`node --version`, `docker --version`) before proceeding to avoid setup issues.
{: .prompt-tip }

## Building the Node.js App
I set up a Node.js app with Express to serve a "Hello, DevOps!" message on port `3000`.

**Create the app** 
In `app.js`:
```javascript
  const express = require('express');
  const app = express();
  app.get('/', (req, res) => res.send('Hello, DevOps!'));
  app.listen(3000, () => console.log('Running on 3000'));
```

**Define Dependencies** 
In `package.json`:
```json
{
  "name": "ci-cd-demo",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.2"
  },
  "scripts": {
    "start": "node app.js"
  }
}
```

**Containerize the App** 
In `Dockerfile`:
```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

I tested locally:
```bash
docker build -t ci-cd-demo .
docker run -p 3000:3000 ci-cd-demo
```

Visiting `http://localhost:3000` displayed “Hello, DevOps!”, confirming the app was ready for automation.

## Setting Up the CI/CD Pipeline with GitHub Actions
I created a GitHub Actions workflow to build and test the Docker image on pushes to the `main` branch. In `.github/workflows/deploy.yml`:
```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker Image
        run: docker build -t ci-cd-demo .
      - name: Test
        run: docker run -d -p 3000:3000 ci-cd-demo && sleep 5 && curl http://localhost:3000
```

This workflow builds the image and tests the app by running the container and verifying the response with curl.

I pushed the code to GitHub:
```bash
git add .
git commit -m "feat(app): add Node.js app with CI/CD pipeline"
git push origin main
```
> Ensure the container has enough startup time before testing—adjust the sleep duration (e.g., 10 seconds) if curl fails with "Connection refused."
{: .prompt-warning }

## Verifying the Pipeline
I checked the pipeline run in the "Actions" tab of my GitHub repository (`https://github.com/yourusername/ci-cd-demo`). The workflow succeeded, with the test step logging "Hello, DevOps!". An initial failure due to a "Connection refused" error was fixed by increasing the sleep duration to 10 seconds. 
```yaml
- name: Test
  run: docker run -d -p 3000:3000 ci-cd-demo && sleep 10 && curl http://localhost:3000
```

## Visualizing the CI/CD Pipeline
I created a diagram using draw.io to illustrate the flow from code push to testing:

![Desktop View](/assets/img/posts/20250407/ci-cd-demo.png){: width="100%" height="auto" }
_Diagram of the CI/CD pipeline_

## Cleanming Up
I stopped local Docker containers after testing:
```bash
docker ps # Find the container ID
docker stop <container-id>
```

> Always stop unused containers to free up system resources and avoid port conflicts.
{: .prompt-tip }

## Conclusion
This CI/CD pipeline automated the build and test process for a Node.js app using GitHub Actions and Docker. Adjusting steps like the sleep duration ensured a stable workflow. It’s a straightforward setup for applying automation in DevOps and SRE practices.