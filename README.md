# BMI & Health Tracker – 3‑Tier Web App with CI/CD & Kubernetes

A three‑tier web application for tracking BMI/BMR and daily calorie needs, containerized with Docker and deployed to Kubernetes with GitHub Actions CI/CD. The stack includes monitoring with Prometheus, Loki, and Grafana.

## Overview

**Tiers**
- **Frontend (Presentation)** – React + Vite SPA, built and served via nginx
- **Backend (Application)** – Node.js + Express API, connects to PostgreSQL
- **Database (Data)** – PostgreSQL storing measurements and health metrics

**DevOps**
- Docker images for frontend and backend
- Kubernetes manifests for all components
- GitHub Actions CI (build/validate) and CD (build, push, deploy)
- Monitoring stack (Prometheus, Loki, Grafana) via Helm

For deep functional and domain details of the app itself, see `AGENT.md`, `CONNECTIVITY.md`, and other docs in this repo.

---

## Project Structure

```text
single-server-3tier-webapp/
  backend/            # Node.js + Express API
  frontend/           # React + Vite SPA
  database/           # Postgres setup & migration scripts
  k8s/                # Kubernetes manifests
  .github/
    workflows/        # GitHub Actions CI/CD pipelines
  README.md
  AGENT.md, CONNECTIVITY.md, ... (additional docs)
```

Key infra files:
- `backend/Dockerfile` – backend container
- `frontend/Dockerfile` – frontend build + nginx runtime
- `k8s/namespace.yaml` – app namespace (`webapp-prod`)
- `k8s/configmap.yaml` – non‑secret config
- `k8s/db-statefulset.yaml` – PostgreSQL StatefulSet + Service
- `k8s/backend-deployment.yaml` – backend Deployment + Service
- `k8s/frontend-deployment.yaml` – frontend Deployment + Service
- `k8s/ingress.yaml` – ingress routing `/` and `/api` paths
- `.github/workflows/ci.yml` – CI pipeline
- `.github/workflows/cd.yml` – CD pipeline

---

## Local Development

### Prerequisites

- Node.js 18+
- npm
- Docker (optional but recommended)
- PostgreSQL (for local DB), or use Docker Postgres

### 1. Backend

```bash
cd backend
npm install

# Create .env based on .env.example
cp .env.example .env
# Edit DATABASE_URL, PORT, etc. if needed

npm run dev
# Backend listens on http://localhost:3000
```

### 2. Frontend

```bash
cd frontend
npm install
npm run dev
# Frontend on http://localhost:5173
# Vite dev server proxies /api → http://localhost:3000
```

Verify locally:
- Frontend: http://localhost:5173
- API health: http://localhost:3000/health
- API base: http://localhost:3000/api

---

## Docker Usage

### Build Images Locally

From project root:

```bash
# Backend
docker build -t bmi-backend:local ./backend

# Frontend
docker build -t bmi-frontend:local ./frontend
```

### Run with Docker (simple example)

```bash
# Run Postgres
docker run -d --name bmi-db \
  -e POSTGRES_DB=bmidb \
  -e POSTGRES_USER=bmi_user \
  -e POSTGRES_PASSWORD=strongpassword \
  -p 5432:5432 postgres:16-alpine

# Run backend (make sure DATABASE_URL matches container)
docker run -d --name bmi-backend \
  -e DATABASE_URL="postgresql://bmi_user:strongpassword@bmi-db:5432/bmidb" \
  --link bmi-db:bmi-db \
  -p 3000:3000 bmi-backend:local

# Run frontend
docker run -d --name bmi-frontend -p 8080:80 bmi-frontend:local
# App available at http://localhost:8080
```

---

## Kubernetes Deployment

These manifests assume:
- Namespace: `webapp-prod`
- DB service name: `bmi-db`
- Backend service name: `webapp-backend` (port 3000)
- Frontend service name: `webapp-frontend` (port 80)
- Ingress class: `nginx`

### 1. Prepare Images

CI/CD uses GitHub Container Registry (GHCR) with images:
- Backend: `ghcr.io/<GITHUB_USER>/<REPO>-backend:TAG`
- Frontend: `ghcr.io/<GITHUB_USER>/<REPO>-frontend:TAG`

**Update the images** in:
- `k8s/backend-deployment.yaml`
- `k8s/frontend-deployment.yaml`

to match your actual GHCR paths (or another registry).

### 2. Apply Core Manifests

With `kubectl` configured for your cluster:

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -n webapp-prod -f k8s/configmap.yaml
kubectl apply -n webapp-prod -f k8s/db-statefulset.yaml
# Secret is usually created by CI; for local/dev you can create manually:
# kubectl apply -n webapp-prod -f k8s/secrets-example.yaml (for testing only)

kubectl apply -n webapp-prod -f k8s/backend-deployment.yaml
kubectl apply -n webapp-prod -f k8s/frontend-deployment.yaml
kubectl apply -n webapp-prod -f k8s/ingress.yaml
```

Check resources:

```bash
kubectl get all -n webapp-prod
kubectl get ingress -n webapp-prod
kubectl logs -n webapp-prod deploy/webapp-backend
```

If you use an ingress controller and DNS/hosts mapping, you should be able to access the app at the host configured in `k8s/ingress.yaml`.

---

## Secret Management

### 1. In Kubernetes

The application expects a Secret named `webapp-secrets` in `webapp-prod` with at least:

- `DB_USER`
- `DB_PASSWORD`
- `DATABASE_URL` (e.g. `postgresql://DB_USER:DB_PASSWORD@bmi-db:5432/bmidb`)
- `JWT_SECRET` (or any app secrets you need)

Backend reads `DATABASE_URL` directly and also inherits config from `k8s/configmap.yaml`.

### 2. From GitHub Actions (CD)

The CD workflow (`.github/workflows/cd.yml`) creates/updates this Secret on each deploy using GitHub Secrets:

Required GitHub Actions secrets:
- `KUBE_CONFIG` – base64‑encoded kubeconfig for the target cluster
- `DB_USER`
- `DB_PASSWORD`
- `DATABASE_URL`
- `JWT_SECRET`

These are injected into the cluster with:

```bash
kubectl -n webapp-prod create secret generic webapp-secrets \
  --from-literal=DB_USER="$DB_USER" \
  --from-literal=DB_PASSWORD="$DB_PASSWORD" \
  --from-literal=DATABASE_URL="$DATABASE_URL" \
  --from-literal=JWT_SECRET="$JWT_SECRET" \
  --dry-run=client -o yaml | kubectl apply -f -
```

Never commit real secrets into Git.

---

## GitHub Actions CI/CD

### CI Workflow (`.github/workflows/ci.yml`)

Runs on pushes and PRs to `main`:
- **Backend job**
  - Checks out code
  - Installs `backend` dependencies
  - Builds backend Docker image
- **Frontend job**
  - Checks out code
  - Installs `frontend` dependencies
  - Builds frontend with Vite
  - Builds frontend Docker image

This validates the app builds and Dockerfiles are healthy.

### CD Workflow (`.github/workflows/cd.yml`)

Runs on pushes to `main` and version tags (`v*.*.*`):
1. Logs into GHCR using `GITHUB_TOKEN`
2. Builds and pushes backend and frontend images with tags:
   - `ghcr.io/<REPO>-backend:<SHORT_SHA>` and `:latest`
   - `ghcr.io/<REPO>-frontend:<SHORT_SHA>` and `:latest`
3. Configures `kubectl` using the `KUBE_CONFIG` secret
4. Creates/updates `webapp-secrets` in the `webapp-prod` namespace
5. Applies manifests from `k8s/`

To enable CD:
1. Push this repo to GitHub.
2. Configure the required Actions secrets.
3. Adjust image names and any registry details as needed.
4. Merge to `main` – the CD pipeline will build and deploy.

---

## Monitoring Stack (Prometheus, Loki, Grafana)

Monitoring is installed at the cluster level (not part of this repo’s manifests). Recommended approach using Helm:

### 1. Prometheus + Grafana

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

This provides:
- Prometheus scraping cluster metrics
- Alertmanager
- Grafana (pre‑wired to Prometheus)

### 2. Loki + Promtail (logs)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set promtail.enabled=true
```

Promtail ships Kubernetes pod logs to Loki. In Grafana, add a **Loki** data source pointing to `http://loki:3100` (or the appropriate service DNS) and explore logs:
- `{app="webapp-backend"}` for backend logs
- `{app="webapp-frontend"}` for frontend logs

### 3. Application Metrics (optional)

If you expose Prometheus metrics from the backend (e.g. `/metrics`), you can:
- Add a named port (e.g. `http-metrics`) on the backend Service
- Create a `ServiceMonitor` in the `monitoring` namespace to have Prometheus scrape those metrics

---

## Security Notes

- Use strong, unique secrets for DB and JWT.
- Never commit `.env` files or real Kubernetes secrets.
- Restrict Kubernetes API and database access to trusted networks.
- Prefer managed databases for production instead of in‑cluster Postgres.

---

## Further Documentation

For more detailed functional, architectural, and deployment docs (including non‑Kubernetes deployment paths), see:
- `AGENT.md` – full technical documentation
- `CONNECTIVITY.md` – three‑tier connectivity details
- `DevOpsReadme.md` and other DevOps‑related docs in this repository
