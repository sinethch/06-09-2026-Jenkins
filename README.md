# MERN Stack E-Commerce Application — Jenkins CI/CD Edition

A production-grade full-stack MERN (MongoDB, Express, React, Node.js) application with automated **Zero-Touch "Push-to-Deploy" CI/CD powered by Jenkins**, **Nginx Web Server & Reverse Proxy**, **Husky Pre-commit Quality Gates**, and **Trivy / npm audit Security Scanning**.

---

## 🚀 DevOps Architecture Overview

```text
 ┌──────────────────────┐
 │  Local Developer     │ ──> git commit (Husky Pre-commit: lint & build validation)
 └──────────┬───────────┘
            │ git push origin main
            ▼
 ┌─────────────────────────────────────────────────────────────┐
 │                     GitHub Repository                       │
 │              github.com/sinethch/06-09-2026-Jenkins         │
 └──────────────────────────────┬──────────────────────────────┘
                                │ GitHub Webhook (HTTP POST)
                                ▼
 ┌─────────────────────────────────────────────────────────────┐
 │          Jenkins CI/CD Server (167.172.77.230:8080)         │
 │                                                             │
 │  Reads → Jenkinsfile (in the repo root)                    │
 │                                                             │
 │  Stage 1: CI  — Backend tests + Frontend build              │
 │  Stage 2: CI  — Docker image verification (local)           │
 │  Stage 3: Sec — npm audit + Trivy container scan            │
 │  Stage 4: CD  — Build & push images → GHCR                 │
 │  Stage 5: CD  — docker compose up (deploy on THIS server)   │
 └──────────────────────────────┬──────────────────────────────┘
                                │ local docker compose up
                                ▼
 ┌─────────────────────────────────────────────────────────────┐
 │           Running Application (same server)                 │
 │                                                             │
 │  ┌──────────────────────────────────────────────────────┐   │
 │  │  Frontend (Nginx) — Port 5173                        │   │
 │  │  Serves React SPA + proxies /api/ to backend         │   │
 │  └────────────────────────────┬─────────────────────────┘   │
 │                               ▼                             │
 │  ┌──────────────────────────────────────────────────────┐   │
 │  │  Backend (Express.js) — Port 5050 → internal 5000    │   │
 │  └────────────────────────────┬─────────────────────────┘   │
 │                               ▼                             │
 │  ┌──────────────────────────────────────────────────────┐   │
 │  │  MongoDB — internal only (port 27017)                │   │
 │  └──────────────────────────────────────────────────────┘   │
 └─────────────────────────────────────────────────────────────┘
```

---

## 🔄 Zero-Touch "Push-to-Deploy" CI/CD (Jenkins)

| Branch | Stages That Run |
|:---|:---|
| `feature/*`, `fix/*`, `hotfix/*` | CI only (test + build + docker verify) |
| `main`, `staging`, `qa` | CI + Security Scan + CD (full deploy) |

### Jenkins Credentials Required

Configure these in **Jenkins → Manage Jenkins → Credentials → Global → Add Credential**:

| Jenkins Credential ID | Type | Description |
|:---|:---|:---|
| `GITHUB_TOKEN` | Secret text | GitHub PAT with `write:packages` + `read:packages` scope |

> No SSH credentials needed — Jenkins deploys directly since it runs on the same server.

---

## 🔗 Pipeline Files

| File | Purpose |
|:---|:---|
| [`Jenkinsfile`](Jenkinsfile) | Main pipeline — replaces all GitHub Actions workflows |
| [`docker-compose.deploy.yml`](docker-compose.deploy.yml) | Production Docker Compose (uses GHCR images) |
| [`docker-compose.yml`](docker-compose.yml) | Local development Docker Compose (builds locally) |

---

## 🛡️ DevOps & Security Tooling

### 1. Nginx Production Web Server & Reverse Proxy
- **Multi-stage Docker build** (`node:20-alpine` builder → `nginx:alpine` runtime).
- **SPA client-side routing fallback**: `try_files $uri $uri/ /index.html;` ensures page refreshes never 404.
- **Internal API reverse proxy**: Requests to `/api/` are forwarded directly to the backend container.
- **Production HTTP security headers**: `X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection`.

### 2. Husky Pre-commit Quality Gates
- Configured in root [`package.json`](package.json) and [`.husky/pre-commit`](.husky/pre-commit).
- Automatically triggers before any `git commit`:
  1. Backend syntax validation (`node --check src/server.js`).
  2. Frontend production build verification (`vite build`).
- Rejects commits if errors or breaking changes are detected.

### 3. Security Pipeline (Jenkins)
- **npm audit**: Scans backend & frontend packages for known security advisories.
- **Trivy Container Security Scanner**: Detects CVEs in Docker images (CRITICAL & HIGH).
- Runs automatically on every push to `main`, `staging`, `qa`.

---

## 💻 Local Development Setup

### Prerequisites
- **Node.js** (v20+)
- **Docker & Docker Compose**

### Install & Run

```bash
# Install root workspace dependencies (Husky)
npm install

# Run backend (dev)
cd backend && npm install && npm run dev

# Run frontend (dev)
cd frontend && npm install && npm run dev

# Run full stack locally with Docker
docker compose up -d --build
```

---

## 🔍 Application Health Verification

- **Frontend UI**: `http://167.172.77.230:5173/`
- **API Health Check**: `http://167.172.77.230:5173/api/health`
- **Jenkins Dashboard**: `http://167.172.77.230:8080`

---

## 📦 Docker Images (GHCR)

Images are pushed to GitHub Container Registry automatically on deploy:

```
ghcr.io/sinethch/06-09-2026-jenkins/backend:main-latest
ghcr.io/sinethch/06-09-2026-jenkins/frontend:main-latest
```
