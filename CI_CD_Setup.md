# CI/CD Setup for BMI & Health Tracker (3‑Tier Web App)

This document explains how the GitHub Actions workflows in `.github/workflows/ci.yml` and `.github/workflows/cd.yml` work for this project.

---

## 1. CI Pipeline (ci.yml)

**File:** `.github/workflows/ci.yml`

### 1.1 Trigger Conditions

```yaml
on:
  push:
    branches: [ main ]
  pull_request:
```

The CI pipeline runs automatically when:
- Code is pushed to the `main` branch, or
- A pull request is opened/updated against any branch.

This ensures all changes going into `main` are validated.

### 1.2 Jobs Overview

The CI workflow defines **two independent jobs**:
- `backend` – validates and builds the Node.js API
- `frontend` – validates and builds the React/Vite SPA

Both jobs run in parallel on `ubuntu-latest` runners.

### 1.3 Backend Job

```yaml
jobs:
  backend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Build Docker image (backend)
        run: docker build -t backend-ci-test .
```

**Step breakdown:**
- **Checkout** – pulls the repository code into the runner.
- **Set up Node.js** – installs Node.js v20.
- **Install dependencies** – runs `npm ci` in the `backend` folder for a clean, reproducible install.
- **Build Docker image (backend)** – runs `docker build` using `backend/Dockerfile` to ensure the backend container image builds successfully.

> Note: Tests/linting can be added here later (e.g. `npm test`, `npm run lint`).

### 1.4 Frontend Job

```yaml
  frontend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Build frontend
        run: npm run build

      - name: Build Docker image (frontend)
        run: docker build -t frontend-ci-test .
```

**Step breakdown:**
- **Checkout** – pulls the repo code.
- **Set up Node.js** – installs Node.js v20.
- **Install dependencies** – `npm ci` in `frontend`.
- **Build frontend** – runs `npm run build` using Vite, generating production assets under `dist/`.
- **Build Docker image (frontend)** – validates the `frontend/Dockerfile` by building a container image.

**Purpose of CI:**
- Ensure JavaScript dependencies install correctly.
- Confirm frontend build succeeds.
- Confirm both Dockerfiles are valid and build without errors.

No images are pushed to a registry in CI – it is purely a validation pipeline.

---

## 2. CD Pipeline (cd.yml)

**File:** `.github/workflows/cd.yml`

### 2.1 Trigger Conditions

```yaml
on:
  push:
    branches: [ main ]
    tags:
      - "v*.*.*"
```

The CD pipeline runs when:
- A commit is pushed to the `main` branch, or
- A tag matching `vX.Y.Z` is pushed (e.g., `v1.0.0`).

This allows both continuous deployment from `main` and versioned releases.

### 2.2 Global Environment

```yaml
env:
  REGISTRY: ghcr.io
```

Defines `REGISTRY` as GitHub Container Registry (GHCR).

### 2.3 Job Overview

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write
      id-token: write
```

The `deploy` job:
- Runs on `ubuntu-latest`.
- Has permissions to read repo contents, write to packages (GHCR), and use OIDC tokens if needed.

### 2.4 Steps Breakdown

#### 2.4.1 Checkout Code

```yaml
- name: Checkout
  uses: actions/checkout@v4
```

Gets the repository source code.

#### 2.4.2 Log in to Container Registry (GHCR)

```yaml
- name: Log in to GHCR
  uses: docker/login-action@v3
  with:
    registry: ${{ env.REGISTRY }}
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

- Authenticates Docker to `ghcr.io`.
- Uses the built‑in `GITHUB_TOKEN` to push images under the current repository’s namespace.

#### 2.4.3 Set Image Tag

```yaml
- name: Set image tag
  id: vars
  run: |
    TAG=${GITHUB_SHA::7}
    echo "TAG=$TAG" >> $GITHUB_OUTPUT
```

- Computes a short tag from the current commit SHA (first 7 chars).
- Exposes it as `steps.vars.outputs.TAG` for later steps.

#### 2.4.4 Build & Push Backend Image

```yaml
- name: Build and push backend image
  uses: docker/build-push-action@v6
  with:
    context: ./backend
    file: ./backend/Dockerfile
    push: true
    tags: |
      ${{ env.REGISTRY }}/${{ github.repository }}-backend:${{ steps.vars.outputs.TAG }}
      ${{ env.REGISTRY }}/${{ github.repository }}-backend:latest
```

- Builds the backend image from `backend/Dockerfile`.
- Pushes to GHCR with:
  - A versioned tag (short SHA)
  - The `latest` tag

Image names look like:
- `ghcr.io/<owner>/<repo>-backend:<short_sha>`
- `ghcr.io/<owner>/<repo>-backend:latest`

#### 2.4.5 Build & Push Frontend Image

```yaml
- name: Build and push frontend image
  uses: docker/build-push-action@v6
  with:
    context: ./frontend
    file: ./frontend/Dockerfile
    push: true
    tags: |
      ${{ env.REGISTRY }}/${{ github.repository }}-frontend:${{ steps.vars.outputs.TAG }}
      ${{ env.REGISTRY }}/${{ github.repository }}-frontend:latest
```

Same logic as backend, but for the frontend image.

#### 2.4.6 Configure kubectl (Kubeconfig from Secret)

```yaml
- name: Set up kubeconfig
  env:
    KUBE_CONFIG_DATA: ${{ secrets.KUBE_CONFIG }}
  run: |
    mkdir -p $HOME/.kube
    echo "$KUBE_CONFIG_DATA" | base64 -d > $HOME/.kube/config
```

- Reads `KUBE_CONFIG` from GitHub Secrets (base64‑encoded kubeconfig).
- Creates `$HOME/.kube/config` so `kubectl` can talk to the target cluster.

> **Important:** You must create a `KUBE_CONFIG` secret in the repository (Actions → Secrets and variables → Actions).

#### 2.4.7 Create/Update Application Secrets in Kubernetes

```yaml
- name: Create/Update app secrets in cluster
  env:
    DB_USER: ${{ secrets.DB_USER }}
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
    JWT_SECRET: ${{ secrets.JWT_SECRET }}
  run: |
    kubectl -n webapp-prod create secret generic webapp-secrets \
      --from-literal=DB_USER="$DB_USER" \
      --from-literal=DB_PASSWORD="$DB_PASSWORD" \
      --from-literal=DATABASE_URL="$DATABASE_URL" \
      --from-literal=JWT_SECRET="$JWT_SECRET" \
      --dry-run=client -o yaml | kubectl apply -f -
```

- Reads the following from GitHub Secrets:
  - `DB_USER`
  - `DB_PASSWORD`
  - `DATABASE_URL`
  - `JWT_SECRET`
- Uses `kubectl` to **create or update** a Secret named `webapp-secrets` in the `webapp-prod` namespace.
- `--dry-run=client -o yaml | kubectl apply -f -` ensures idempotency (works for both create/update).

This Secret is consumed by the backend deployment as environment variables (e.g. `DATABASE_URL`).

#### 2.4.8 Apply Kubernetes Manifests

```yaml
- name: Apply Kubernetes manifests
  run: |
    kubectl apply -f k8s/namespace.yaml
    kubectl apply -n webapp-prod -f k8s/configmap.yaml
    kubectl apply -n webapp-prod -f k8s/db-statefulset.yaml
    kubectl apply -n webapp-prod -f k8s/backend-deployment.yaml
    kubectl apply -n webapp-prod -f k8s/frontend-deployment.yaml
    kubectl apply -n webapp-prod -f k8s/ingress.yaml
```

- Ensures the namespace exists.
- Applies configuration and deployments for:
  - ConfigMap (`webapp-config`)
  - PostgreSQL StatefulSet (`bmi-db`)
  - Backend Deployment + Service (`webapp-backend`)
  - Frontend Deployment + Service (`webapp-frontend`)
  - Ingress (`webapp-ingress`)

This effectively **deploys the latest images** to the Kubernetes cluster.

---

## 3. Required GitHub Secrets

To make the CD pipeline work, configure these in
**GitHub → Repository → Settings → Secrets and variables → Actions**:

- `KUBE_CONFIG` – base64‑encoded kubeconfig for the target Kubernetes cluster.
- `DB_USER` – database username (`bmi_user` or similar).
- `DB_PASSWORD` – database user password.
- `DATABASE_URL` – connection string used by the backend, e.g.:
  - `postgresql://bmi_user:strongpassword@bmi-db:5432/bmidb`
- `JWT_SECRET` – secret key used by the backend for signing tokens (or other app secrets).

> Never commit real credentials or kubeconfig files to Git. Always use GitHub Secrets.

---

## 4. How to Use These Workflows

### 4.1 Development Flow

1. Open a feature branch and push commits.
2. Create a Pull Request to `main`.
3. **CI (ci.yml)** runs automatically:
   - Builds backend and frontend
   - Validates Docker builds
4. Fix any CI issues and merge PR into `main`.

### 4.2 Deployment Flow

Once changes are merged into `main`:

1. **CD (cd.yml)** is triggered by the push to `main`.
2. CD pipeline:
   - Builds & pushes Docker images to GHCR
   - Configures `kubectl` with `KUBE_CONFIG`
   - Updates `webapp-secrets` in `webapp-prod`
   - Applies Kubernetes manifests
3. New version is rolled out to the cluster.

For a tagged release:
1. Create an annotated tag, e.g. `v1.0.0`, and push it.
2. CD runs the same pipeline, but you can use the tag in your release process and dashboards.

---

## 5. Extending the Pipelines

Some common extensions you can add later:

- **Tests & Linting in CI**
  - `npm test` and `npm run lint` for both backend and frontend jobs.
- **Deploy to Staging vs Production**
  - Use different branches or environments (`staging`, `prod`) with separate clusters or namespaces.
- **Blue‑Green or Canary Deployments**
  - Use Kubernetes strategies and additional manifests to support progressive rollout.
- **Notifications**
  - Integrate with Slack/Teams/email for failed CI/CD runs.

These workflows give you a solid foundation for automated build, containerization, and deployment of the 3‑tier application.
