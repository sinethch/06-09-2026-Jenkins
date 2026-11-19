# Jenkins Pipeline and Deployment Guide

This repository uses Jenkins for CI, security reporting, image publishing, and deployment. The pipeline stages are defined in [`../Jenkinsfile`](../Jenkinsfile). The GitHub Actions workflow files are not part of this pipeline.

## Branch behavior

The Jenkinsfile runs checkout, backend checks, frontend builds, and Docker image verification for the branch checked out by the Jenkins job. The dependency audit, Trivy reports, GHCR publish, and Docker Compose deployment stages run only for `main`, `staging`, and `qa`.

The pipeline does not declare its own webhook or polling trigger. Configure the repository connection and trigger in Jenkins. A multibranch pipeline can discover branch jobs; a webhook or polling schedule must be configured separately if automatic runs are required.

## Jenkins prerequisites

- A Jenkins agent with Docker access, Git, and permission to run Docker builds and Compose deployments.
- Node.js 20 available to the pipeline's temporary `node:20-alpine` containers.
- A Jenkins Secret Text credential named `GITHUB_TOKEN`. The token needs permission to publish packages to GHCR and to pull the private images during deployment.
- Writable access to the configured deployment directory (`/var/lib/jenkins/deploy/ecommerce` in the current Jenkinsfile).
- The Compose plugin and sufficient host disk, memory, and network access for MongoDB and the application containers.

The optional Jenkins controller in [`docker-compose.jenkins.yml`](../docker-compose.jenkins.yml) mounts the host Docker socket. This grants pipeline jobs broad control of the Docker host. Run trusted pipeline code only and restrict who can change or trigger it.

## Image tags and deployment

For the `main`, `staging`, and `qa` branches, Jenkins builds and publishes backend and frontend images to GHCR. Each image receives a branch-and-commit tag (`<branch>-<short-sha>`) and a branch alias (`<branch>-latest`).

The deploy stage copies [`../docker-compose.deploy.yml`](../docker-compose.deploy.yml) to the deployment directory, authenticates to GHCR, pulls the commit-specific backend and frontend images, then runs `docker compose up -d --remove-orphans`. The Compose stack includes MongoDB with a named data volume. The frontend is published on port `5173` by default and the backend on port `5050`.

The pipeline prints running container details but does not make an HTTP request to confirm the application is healthy after deployment. Check container status and test the frontend and `/api/health` endpoint separately when verifying a deployment.

## Security scan behavior

The pipeline runs `npm audit` for backend and frontend dependencies and Trivy against the locally built application images on `main`, `staging`, and `qa`. These commands are configured to report findings without failing the pipeline. Treat the reports as visibility, not as a blocking security gate.

## Local development

For a local source build, run from the repository root:

```bash
docker compose up -d --build
```

For the image-based deployment stack, provide the image and environment variables consumed by `docker-compose.deploy.yml`, then run:

```bash
docker compose -f docker-compose.deploy.yml pull
docker compose -f docker-compose.deploy.yml up -d --remove-orphans
```
