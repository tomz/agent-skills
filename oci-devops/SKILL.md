---
name: oci-devops
description: OCI DevOps service — build/deploy pipelines, OCIR container registry, GitHub Actions integration, Artifact Registry
license: MIT
version: 1.0.0
allowed-tools:
  - shell
  - read_file
  - write_file
  - glob
  - grep
---

# OCI DevOps

## OCI DevOps Service Overview

OCI DevOps covers the full CI/CD lifecycle:
- **Code Repositories** — managed Git (or mirror GitHub/GitLab/Bitbucket)
- **Build Pipelines** — compile, test, build container images
- **Artifact Registry / OCIR** — store build outputs (containers, generic artifacts)
- **Deployment Pipelines** — deploy to OKE, instances, Functions, or OCI DevOps environments

## Setup: DevOps Project

```bash
# Create a notification topic (required for DevOps project)
TOPIC_ID=$(oci ons topic create \
  --compartment-id $C \
  --name "devops-notifications" \
  --query 'data."topic-id"' --raw-output)

# Create DevOps project
PROJECT_ID=$(oci devops project create \
  --compartment-id $C \
  --name "my-app" \
  --notification-config '{"topicId":"'$TOPIC_ID'"}' \
  --description "My application CI/CD" \
  --query 'data.id' --raw-output)
```

## Code Repositories

```bash
# Create managed Git repo
REPO_ID=$(oci devops repository create \
  --project-id $PROJECT_ID \
  --name "app-repo" \
  --repository-type "HOSTED" \
  --query 'data.id' --raw-output)

# Get clone URL
oci devops repository get --repository-id $REPO_ID \
  --query 'data."http-url"' --raw-output

# Clone using OCI token auth
git clone https://<tenancy>/<username>@devops.scmservice.us-ashburn-1.oci.oraclecloud.com/namespaces/<namespace>/projects/my-app/repositories/app-repo

# Mirror external repo (GitHub)
oci devops repository create \
  --project-id $PROJECT_ID \
  --name "app-repo-mirror" \
  --repository-type "MIRRORED" \
  --mirror-repository-config '{
    "repositoryUrl": "https://github.com/myorg/myapp",
    "triggerSchedule": {"scheduleType":"DEFAULT"},
    "connectorId": "<external-connection-ocid>"
  }'
```

## Build Pipelines

### Build Pipeline Spec (build_spec.yaml)

Place this file in your repository root:

```yaml
version: 0.1
updated: 2026-04-24
component: build
timeoutInSeconds: 600

env:
  variables:
    APP_VERSION: "1.0.0"
  exportedVariables:
    - IMAGE_TAG

steps:
  - type: Command
    name: "Run unit tests"
    command: |
      cd app/
      pip install -r requirements.txt
      pytest tests/ -v

  - type: Command
    name: "Build container image"
    command: |
      export IMAGE_TAG="${APP_VERSION}-${OCI_BUILD_RUN_ID}"
      docker build -t myapp:${IMAGE_TAG} .
      echo "IMAGE_TAG=${IMAGE_TAG}" >> $OCI_EXPORTED_VARS

  - type: Command
    name: "Push to OCIR"
    command: |
      docker tag myapp:${IMAGE_TAG} ${OCIR_REGION}.ocir.io/${NAMESPACE}/${REPO}:${IMAGE_TAG}
      echo "${OCIR_AUTH_TOKEN}" | docker login ${OCIR_REGION}.ocir.io \
        -u "${TENANCY_NAMESPACE}/${OCI_USERNAME}" --password-stdin
      docker push ${OCIR_REGION}.ocir.io/${NAMESPACE}/${REPO}:${IMAGE_TAG}

outputArtifacts:
  - name: app-image
    type: DOCKER_IMAGE
    location: ${OCIR_REGION}.ocir.io/${NAMESPACE}/${REPO}:${IMAGE_TAG}
```

### Create Build Pipeline
```bash
PIPELINE_ID=$(oci devops build-pipeline create \
  --project-id $PROJECT_ID \
  --display-name "app-build" \
  --query 'data.id' --raw-output)

# Add build stage
oci devops build-pipeline-stage create-build-stage \
  --build-pipeline-id $PIPELINE_ID \
  --display-name "build-and-test" \
  --build-spec-file "build_spec.yaml" \
  --image "OL7_X86_64_STANDARD_10" \
  --primary-build-source '{"connectionType":"DEVOPS_CODE_REPOSITORY","repositoryId":"'$REPO_ID'","repositoryUrl":"<url>","branch":"main","name":"app-source"}' \
  --stage-execution-timeout-in-seconds 600 \
  --build-pipeline-stage-predecessor-collection '{"items":[{"id":"'$PIPELINE_ID'"}]}'

# Add deliver artifacts stage
oci devops build-pipeline-stage create-deliver-artifact-stage \
  --build-pipeline-id $PIPELINE_ID \
  --display-name "push-artifact" \
  --deliver-artifact-collection '{"items":[{"artifactId":"<artifact-ocid>","artifactName":"app-image"}]}' \
  --build-pipeline-stage-predecessor-collection '{"items":[{"id":"<build-stage-ocid>"}]}'

# Manually trigger build run
oci devops build-run create \
  --build-pipeline-id $PIPELINE_ID \
  --display-name "manual-run-$(date +%Y%m%d)" \
  --commit-info '{"repositoryUrl":"<url>","repositoryBranch":"main"}'

# Watch build run status
oci devops build-run get --build-run-id $BUILD_RUN_ID \
  --query 'data."lifecycle-state"' --raw-output
```

## Deployment Pipelines

### Environments
```bash
# OKE environment
oci devops deploy-environment create-oke-cluster-environment \
  --project-id $PROJECT_ID \
  --display-name "prod-oke" \
  --cluster-id $OKE_CLUSTER_ID

# Instance group environment
oci devops deploy-environment create-compute-instance-group-environment \
  --project-id $PROJECT_ID \
  --display-name "prod-instances" \
  --compute-instance-group-selectors '{"items":[{"selectorType":"INSTANCE_IDS","computeInstanceIds":["'$INST_ID'"]}]}'
```

### Create Deploy Pipeline
```bash
DEPLOY_PIPELINE_ID=$(oci devops deploy-pipeline create \
  --project-id $PROJECT_ID \
  --display-name "app-deploy" \
  --query 'data.id' --raw-output)

# Rolling deploy to OKE stage
oci devops deploy-stage create-oke-helm-stage \
  --deploy-pipeline-id $DEPLOY_PIPELINE_ID \
  --display-name "deploy-to-oke" \
  --oke-cluster-deploy-environment-id <env-ocid> \
  --helm-chart-deploy-artifact-id <artifact-ocid> \
  --release-name "myapp" \
  --namespace "production" \
  --rollback-policy '{"policyType":"AUTOMATED_STAGE_ROLLBACK_POLICY"}' \
  --deploy-stage-predecessor-collection '{"items":[{"id":"'$DEPLOY_PIPELINE_ID'"}]}'
```

## Triggers (Auto-run on push)

```bash
# Trigger on push to main branch
oci devops trigger create-devops-code-repository-trigger \
  --project-id $PROJECT_ID \
  --display-name "push-to-main" \
  --repository-id $REPO_ID \
  --actions '[{
    "type": "TRIGGER_BUILD_PIPELINE",
    "filter": {
      "triggerSource": "DEVOPS_CODE_REPOSITORY",
      "events": ["PUSH"],
      "include": {"headRef": "main"}
    },
    "buildPipelineId": "'$PIPELINE_ID'"
  }]'

# GitHub trigger
oci devops trigger create-github-trigger \
  --project-id $PROJECT_ID \
  --display-name "github-push" \
  --actions '[{
    "type": "TRIGGER_BUILD_PIPELINE",
    "filter": {"triggerSource":"GITHUB","events":["PUSH"],"include":{"headRef":"main"}},
    "buildPipelineId": "'$PIPELINE_ID'"
  }]'
```

## OCIR — Oracle Container Image Registry

```bash
# Authenticate to OCIR (use auth token, NOT account password)
docker login iad.ocir.io \
  -u "${TENANCY_NAMESPACE}/${USERNAME}" \
  -p "${OCI_AUTH_TOKEN}"

# Push image
docker tag myapp:latest iad.ocir.io/${TENANCY_NAMESPACE}/myrepo/myapp:latest
docker push iad.ocir.io/${TENANCY_NAMESPACE}/myrepo/myapp:latest

# Create repository
oci artifacts container repository create \
  --compartment-id $C \
  --display-name "myrepo/myapp" \
  --is-public false

# List repositories
oci artifacts container repository list --compartment-id $C --all --output table

# List images in repository
oci artifacts container image list \
  --compartment-id $C \
  --repository-name "myrepo/myapp" \
  --all --output table

# Scan image for vulnerabilities
oci vulnerability-scanning container-scan-recipe create \
  --compartment-id $C \
  --display-name "container-scan" \
  --scan-settings '{"scanLevel":"STANDARD"}'

# Delete old images (cleanup)
oci artifacts container image delete \
  --image-id <image-ocid> --force

# Get region-specific OCIR endpoint
# us-ashburn-1 → iad.ocir.io
# eu-frankfurt-1 → fra.ocir.io
# ap-tokyo-1 → nrt.ocir.io
# us-phoenix-1 → phx.ocir.io
```

### Image Signing (Notary / Cosign)
```bash
# Sign image with OCI Vault key
oci artifacts container image-signature sign-upload \
  --compartment-id $C \
  --image-id <image-ocid> \
  --kms-key-id $KEY_ID \
  --kms-key-version-id $KEY_VERSION_ID \
  --signing-algorithm "SHA_256_RSA_PKCS_PSS" \
  --message '{"description":"Signed by CI pipeline","buildRunId":"'$BUILD_RUN_ID'"}'
```

## Artifact Registry (Generic Artifacts)

```bash
# Create artifact repository
ARTIFACT_REPO_ID=$(oci artifacts repository create \
  --compartment-id $C \
  --display-name "build-artifacts" \
  --repository-type "GENERIC" \
  --is-immutable false \
  --query 'data.id' --raw-output)

# Upload artifact
oci artifacts generic artifact upload-by-path \
  --repository-id $ARTIFACT_REPO_ID \
  --artifact-path "myapp/binaries" \
  --artifact-version "1.0.0" \
  --content-body ./myapp-1.0.0.tar.gz

# Download artifact
oci artifacts generic artifact download-by-path \
  --repository-id $ARTIFACT_REPO_ID \
  --artifact-path "myapp/binaries" \
  --artifact-version "1.0.0" \
  --file ./myapp-1.0.0.tar.gz
```

## GitHub Actions for OCI

```yaml
# .github/workflows/deploy.yml
name: Deploy to OCI

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure OCI CLI
        uses: oracle-actions/configure-kubectl-oke@v1.3.2
        with:
          cluster: ${{ secrets.OKE_CLUSTER_OCID }}

      - name: Login to OCIR
        uses: docker/login-action@v3
        with:
          registry: iad.ocir.io
          username: ${{ secrets.OCIR_USERNAME }}   # tenancy/user
          password: ${{ secrets.OCIR_TOKEN }}       # auth token

      - name: Build & Push
        run: |
          docker build -t iad.ocir.io/${{ secrets.TENANCY_NS }}/myrepo/app:${{ github.sha }} .
          docker push iad.ocir.io/${{ secrets.TENANCY_NS }}/myrepo/app:${{ github.sha }}

      - name: Deploy to OKE
        run: |
          kubectl set image deployment/myapp \
            app=iad.ocir.io/${{ secrets.TENANCY_NS }}/myrepo/app:${{ github.sha }}
          kubectl rollout status deployment/myapp

      - name: Trigger OCI DevOps Deploy Pipeline
        uses: oracle-actions/run-oci-cli-command@v1.1.1
        with:
          command: devops deployment create --deploy-pipeline-id ${{ secrets.DEPLOY_PIPELINE_ID }}
          silent: false
```

## Resource Manager for CI/CD

```bash
# Update and apply a Resource Manager stack as part of CI (Terraform)
zip -r stack.zip *.tf

oci resource-manager stack update \
  --stack-id $STACK_ID \
  --config-source '{"configSourceType":"ZIP_UPLOAD","zipFileBase64Encoded":"'$(base64 -w0 stack.zip)'"}'

JOB_ID=$(oci resource-manager job create-apply-job \
  --stack-id $STACK_ID \
  --execution-plan-strategy AUTO_APPROVED \
  --query 'data.id' --raw-output)

# Poll until done
while true; do
  STATE=$(oci resource-manager job get --job-id $JOB_ID \
    --query 'data."lifecycle-state"' --raw-output)
  echo "State: $STATE"
  [[ "$STATE" == "SUCCEEDED" || "$STATE" == "FAILED" ]] && break
  sleep 15
done
```

## Gotchas & Tips

- **Auth tokens for OCIR** — Docker login to OCIR uses OCI Auth Tokens (IAM → User → Auth Tokens), NOT your console password. Max 2 tokens per user.
- **OCIR username format** — `<tenancy-namespace>/<iam-username>` for native users, or `<tenancy-namespace>/oracleidentitycloudservice/<email>` for IDCS-federated users.
- **Build image options** — `OL7_X86_64_STANDARD_10` is Oracle Linux 7 with Docker. Use `OL8_X86_64_STANDARD_10` for OL8. Custom build runners (BYO) are supported.
- **Build pipeline exported variables** — variables exported in `build_spec.yaml` must be declared in `exportedVariables` section AND written to `$OCI_EXPORTED_VARS` file.
- **Deploy approval stages** — add a manual approval stage before prod deployments: `create-manual-approval-stage`. Approvers are notified via the notification topic.
- **OKE deploy strategy** — use `BLUE_GREEN_DEPLOYMENT_STRATEGY` or `CANARY_DEPLOYMENT_STRATEGY` for zero-downtime deployments to OKE.
- **IAM policy for DevOps** — pipelines run as `devopspipeline` service. Add policy: `Allow service devops to manage all-resources in compartment $C` for full pipeline capabilities.
- **Artifact immutability** — set `--is-immutable true` for release registries. Immutable repos prevent overwriting existing artifact versions — critical for reproducibility.
