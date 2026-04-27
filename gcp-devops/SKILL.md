---
name: gcp-devops
description: GCP DevOps — Cloud Build, Artifact Registry, Cloud Deploy, GitHub Actions, Source Repositories, Skaffold
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, glob, grep
---

# GCP DevOps Skill

Build, package, and deploy applications on GCP using fully managed CI/CD.

---

## Cloud Build

Cloud Build runs CI/CD pipelines in managed ephemeral containers.

### cloudbuild.yaml Structure

```yaml
# cloudbuild.yaml
steps:
  # Step 1: Run tests
  - name: 'python:3.11'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        pip install -r requirements.txt
        pytest tests/ -v

  # Step 2: Build Docker image
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'us-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'
      - '-t'
      - 'us-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:latest'
      - '.'

  # Step 3: Push to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '--all-tags', 'us-docker.pkg.dev/$PROJECT_ID/my-repo/my-app']

  # Step 4: Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'gcloud'
    args:
      - 'run'
      - 'deploy'
      - 'my-service'
      - '--image=us-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'
      - '--region=us-central1'

# Build artifacts to pass between steps
artifacts:
  images:
    - 'us-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'

# Global timeout
timeout: '1200s'

# Cache configuration (faster builds)
options:
  machineType: 'E2_HIGHCPU_8'
  logging: CLOUD_LOGGING_ONLY
  substitutionOption: ALLOW_LOOSE
```

### Available Substitutions

| Variable | Value |
|---------|-------|
| `$PROJECT_ID` | GCP project ID |
| `$BUILD_ID` | Unique build ID |
| `$COMMIT_SHA` | Full git commit SHA |
| `$SHORT_SHA` | First 7 chars of commit SHA |
| `$REPO_NAME` | Repository name |
| `$BRANCH_NAME` | Git branch |
| `$TAG_NAME` | Git tag (if triggered by tag) |
| `$REF_NAME` | Branch or tag name |

### Running Builds

```bash
# Submit a build manually
gcloud builds submit . \
  --config=cloudbuild.yaml \
  --substitutions=_ENV=staging,_VERSION=v1.2.3

# Submit with a specific tag
gcloud builds submit . \
  --tag=us-docker.pkg.dev/my-project/my-repo/my-app:latest

# List recent builds
gcloud builds list --limit=10

# Stream logs of a build
gcloud builds log BUILD_ID --stream

# Cancel a running build
gcloud builds cancel BUILD_ID
```

### Triggers

```bash
# Create a trigger on push to main
gcloud builds triggers create github \
  --repo-name=my-repo \
  --repo-owner=myorg \
  --branch-pattern='^main$' \
  --build-config=cloudbuild.yaml \
  --name=deploy-on-main

# Create a trigger on PR
gcloud builds triggers create github \
  --repo-name=my-repo \
  --repo-owner=myorg \
  --pull-request-pattern='^main$' \
  --build-config=cloudbuild-pr.yaml \
  --name=pr-checks \
  --comment-control=COMMENTS_ENABLED

# Create a trigger on tag push
gcloud builds triggers create github \
  --repo-name=my-repo \
  --repo-owner=myorg \
  --tag-pattern='^v[0-9]+\.[0-9]+\.[0-9]+$' \
  --build-config=cloudbuild-release.yaml \
  --name=release-on-tag

# List triggers
gcloud builds triggers list

# Run a trigger manually
gcloud builds triggers run my-trigger \
  --branch=main
```

### Private Worker Pools

```bash
# Create a private pool (for access to VPC resources)
gcloud builds worker-pools create my-pool \
  --region=us-central1 \
  --peered-network=projects/my-project/global/networks/my-vpc \
  --worker-machine-type=e2-highcpu-8 \
  --worker-disk-size=100GB

# Use the pool in cloudbuild.yaml
# options:
#   workerPool: projects/my-project/locations/us-central1/workerPools/my-pool
```

---

## Artifact Registry

The recommended private registry for Docker images, language packages (npm, pip, Maven), and more.

```bash
# Create a repository
gcloud artifacts repositories create my-repo \
  --repository-format=docker \
  --location=us \
  --description="Docker images for my-app"

# Create a Python repo
gcloud artifacts repositories create my-python-repo \
  --repository-format=python \
  --location=us-central1

# Configure Docker auth
gcloud auth configure-docker us-docker.pkg.dev

# Push an image
docker tag my-app:latest us-docker.pkg.dev/my-project/my-repo/my-app:latest
docker push us-docker.pkg.dev/my-project/my-repo/my-app:latest

# List images
gcloud artifacts docker images list us-docker.pkg.dev/my-project/my-repo/my-app

# List tags
gcloud artifacts docker tags list us-docker.pkg.dev/my-project/my-repo/my-app

# Delete old images (cleanup)
gcloud artifacts docker images delete \
  us-docker.pkg.dev/my-project/my-repo/my-app:old-tag \
  --delete-tags

# Set up cleanup policy (delete untagged images older than 30 days)
gcloud artifacts repositories set-cleanup-policies my-repo \
  --location=us \
  --policy=cleanup-policy.json
```

```json
// cleanup-policy.json
[
  {
    "name": "delete-untagged",
    "action": {"type": "Delete"},
    "condition": {
      "tagState": "UNTAGGED",
      "olderThan": "720h"
    }
  }
]
```

---

## Cloud Deploy

Managed continuous delivery for GKE, Cloud Run, and GCE.

### Delivery Pipeline

```yaml
# clouddeploy.yaml
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: my-app-pipeline
serialPipeline:
  stages:
    - targetId: staging
      profiles:
        - staging
    - targetId: prod
      profiles:
        - prod
      strategy:
        canary:
          runtimeConfig:
            cloudRun:
              automaticTrafficControl: true
          canaryDeployment:
            percentages: [10, 50]
            verify: true
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: staging
run:
  location: projects/my-project/locations/us-central1
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod
  requireApproval: true
run:
  location: projects/my-project/locations/us-central1
```

```bash
# Apply the pipeline and targets
gcloud deploy apply --file=clouddeploy.yaml --region=us-central1 --project=my-project

# Create a release (starts a delivery)
gcloud deploy releases create release-v1-2-3 \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1 \
  --images=my-app=us-docker.pkg.dev/my-project/my-repo/my-app:v1.2.3

# List releases
gcloud deploy releases list --delivery-pipeline=my-app-pipeline --region=us-central1

# Promote a release to the next stage
gcloud deploy releases promote \
  --release=release-v1-2-3 \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1

# Approve a rollout (for targets with requireApproval: true)
gcloud deploy rollouts approve my-app-pipeline-20240101-001 \
  --release=release-v1-2-3 \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1

# Roll back a deployment
gcloud deploy rollbacks create \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1 \
  --release=release-v1-2-2
```

---

## GitHub Actions for GCP

### Authentication (Workload Identity — no keys)

```yaml
# .github/workflows/deploy.yml
name: Deploy to Cloud Run

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write  # Required for OIDC

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/providers/github-provider
          service_account: deploy-sa@my-project.iam.gserviceaccount.com

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      - name: Configure Docker auth
        run: gcloud auth configure-docker us-docker.pkg.dev --quiet

      - name: Build and push
        run: |
          docker build -t us-docker.pkg.dev/my-project/my-repo/my-app:${{ github.sha }} .
          docker push us-docker.pkg.dev/my-project/my-repo/my-app:${{ github.sha }}

      - name: Deploy to Cloud Run
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: my-service
          region: us-central1
          image: us-docker.pkg.dev/my-project/my-repo/my-app:${{ github.sha }}
```

### Other Useful Actions

```yaml
# Deploy to GKE
- uses: google-github-actions/get-gke-credentials@v2
  with:
    cluster_name: my-cluster
    location: us-central1

- run: kubectl apply -f k8s/

# Upload to GCS
- uses: google-github-actions/upload-cloud-storage@v2
  with:
    path: ./dist
    destination: my-bucket/releases/${{ github.sha }}

# Cloud Build
- run: |
    gcloud builds submit . \
      --config=cloudbuild.yaml \
      --substitutions=SHORT_SHA=${{ github.sha }}
```

---

## Skaffold Integration

Skaffold handles the build-tag-push-deploy cycle for local and CI.

```yaml
# skaffold.yaml
apiVersion: skaffold/v4beta6
kind: Config
metadata:
  name: my-app

build:
  artifacts:
    - image: us-docker.pkg.dev/my-project/my-repo/my-app
      docker:
        dockerfile: Dockerfile
  googleCloudBuild:
    projectId: my-project

deploy:
  cloudrun:
    projectid: my-project
    region: us-central1
```

```bash
# Local dev (builds locally, deploys to GKE/Cloud Run)
skaffold dev

# One-shot build and deploy
skaffold run --profile=staging

# Build only
skaffold build --file-output=build.json

# Deploy from pre-built artifacts
skaffold deploy --build-artifacts=build.json
```

---

## Source Repositories (Cloud Source Repos)

```bash
# Create a repo
gcloud source repos create my-repo

# Clone it
gcloud source repos clone my-repo --project=my-project

# Mirror a GitHub repo (to use Cloud Build triggers on it)
# Use Cloud Console: Source Repositories → Add Repository → Mirror from GitHub
```

---

## Guardrails

- **Use Workload Identity Federation** in GitHub Actions — never store SA keys as GitHub secrets.
- **Always pin `actions/checkout` and `google-github-actions/*` to version tags** (not `@main`).
- **Cloud Build service account** (`PROJECT_NUMBER@cloudbuild.gserviceaccount.com`) has editor by default — restrict it.
- **Use Cloud Deploy for prod** instead of `gcloud run deploy` directly — it gives you audit trail, approvals, and rollback.
- **Tag Docker images with commit SHA**, not just `latest` — makes rollback and auditing possible.
- **Cache pip/npm in Cloud Build** using GCS-backed caching to speed up builds.
- **Cloud Deploy `requireApproval: true`** on production targets is strongly recommended.
- **Build artifacts should be immutable** — never overwrite an existing SHA-tagged image.
