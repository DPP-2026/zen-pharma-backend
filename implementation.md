# End-to-End Project Setup Guide

Follow these steps **in order**. Each step depends on the previous one being complete.

**Repositories involved**

| Repository | Role |
|------------|------|
| `zen-infra` | Terraform — provisions EKS, ECR, IAM, ArgoCD |
| `zen-pharma-backend` | Backend microservices + CI pipelines |
| `zen-pharma-frontend` | React frontend + CI pipeline |
| `zen-gitops` | Helm values — ArgoCD reads from here |

**Placeholders** — replace these throughout:

- `YOUR_GITHUB_USERNAME` — your GitHub username or org
- `YOUR_AWS_ACCOUNT_ID` — 12-digit AWS account ID
- `YOUR_REGION` — e.g. `us-east-1`

---

## Step 1 — Provision infrastructure via zen-infra

> This creates EKS, ECR repositories, IAM OIDC role, and networking.

1. Fork or clone `zen-infra` to your GitHub account.
2. Set the required secrets in **`zen-infra` → Settings → Secrets and variables → Actions → Secrets**:

   | Secret | Value |
   |--------|-------|
   | `AWS_ACCESS_KEY_ID` | Your IAM user access key |
   | `AWS_SECRET_ACCESS_KEY` | Your IAM user secret key |

3. Go to **Actions → Terraform Infrastructure → Run workflow**.
4. Select:
   - **Environment:** `dev`
   - **Action:** `apply`
5. Click **Run workflow** and wait for it to complete (≈ 15–20 min).
6. From the workflow logs, note:
   - EKS cluster name
   - AWS Account ID
   - ECR registry URL (`YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com`)

> **Verify:** Go to AWS Console → EKS — your cluster should be `Active`. Go to ECR — you should see one repository per microservice (`api-gateway`, `auth-service`, `drug-catalog-service`, `inventory-service`, `manufacturing-service`, `notification-service`, `qc-service`, `supplier-service`, `pharma-ui`).

---

## Step 1b — Personalise zen-gitops for your AWS account

> Do this **before** running the bootstrap scripts. ArgoCD will start syncing `zen-gitops` immediately after Step 2, so image URLs and DB endpoints must point to your account first.

The `envs/dev/*.yaml` files are pre-populated with the instructor's AWS account ID and RDS instance identifier. Leave them as-is and ArgoCD will produce `ImagePullBackOff` errors because it tries to pull images from an ECR registry you don't own.

### 1. Replace the AWS account ID

Every `image.repository` and the IAM role ARN in `values-api-gateway.yaml` contain the placeholder account ID. Replace it with yours (from Step 1 logs or):

```bash
aws sts get-caller-identity --query Account --output text
```

Run from the root of your forked `zen-gitops` repo:

```bash
# macOS/BSD sed (note the '' after -i)
find envs/ -name "*.yaml" -exec sed -i '' 's/516209541629/YOUR_AWS_ACCOUNT_ID/g' {} +

# Linux sed
find envs/ -name "*.yaml" -exec sed -i 's/516209541629/YOUR_AWS_ACCOUNT_ID/g' {} +
```

### 2. Replace the RDS instance identifier

Every `DB_HOST` env var contains the instructor's RDS instance identifier. Find yours in **AWS Console → RDS → Databases → your instance → Endpoint** — it is the subdomain prefix before `.us-east-1.rds.amazonaws.com`.

```bash
# macOS/BSD sed
find envs/ -name "*.yaml" -exec sed -i '' 's/cyrywaguk6v4/YOUR_RDS_ID/g' {} +

# Linux sed
find envs/ -name "*.yaml" -exec sed -i 's/cyrywaguk6v4/YOUR_RDS_ID/g' {} +
```

### 3. Verify and push

```bash
# Should return no output (no instructor values left)
grep -r "516209541629\|cyrywaguk6v4" envs/

git add envs/
git commit -m "chore(envs/dev): replace instructor account ID and RDS endpoint"
git push
```

Once pushed, CI will overwrite image tags with your ECR images on every build and ArgoCD syncs will succeed.

---

## Step 2 — Run the bootstrap scripts (zen-infra/scripts/)

Run these four scripts **in order** from your local machine after the Terraform apply completes. Each script prompts for required values — nothing is hardcoded.

```bash
# First, update your kubeconfig to point at the new cluster
aws eks update-kubeconfig --name <cluster-name> --region YOUR_REGION
```

### Script 01 — Install Kubernetes prerequisites
```bash
bash scripts/01-install-prerequisites.sh
```
Installs: NGINX Ingress Controller, ArgoCD, External Secrets Operator, metrics-server.

### Script 02 — Bootstrap ArgoCD
```bash
bash scripts/02-bootstrap-argocd.sh
```
Registers `zen-gitops` in ArgoCD, creates the `pharma` AppProject, deploys all ArgoCD Application manifests.

### Script 03 — Configure External Secrets
```bash
bash scripts/03-setup-external-secrets.sh
```
Creates `ClusterSecretStore` and `ExternalSecrets` so pods pull DB credentials and JWT secret from AWS Secrets Manager via IRSA.

### Script 04 — Verify deployment
```bash
bash scripts/04-verify-deployment.sh
```
Health-checks pods, ArgoCD apps, External Secrets, services/ingress, and HTTP endpoints. **All checks must pass before proceeding.**

---

## Step 3 — Configure GitHub repository secrets and variables

Do this for **both** `zen-pharma-backend` and `zen-pharma-frontend`.

### zen-pharma-backend

**Settings → Secrets and variables → Actions → Secrets**

| Secret | Required | Value |
|--------|----------|-------|
| `AWS_ACCOUNT_ID` | Yes | Your 12-digit AWS account ID |
| `GITOPS_TOKEN` | Yes | GitHub PAT with `contents: write` + PR permissions on `YOUR_GITHUB_USERNAME/zen-gitops` |
| `NVD_API_KEY` | Optional | NIST NVD API key — see [how to get it](#how-to-get-nvd_api_key) below |

**Settings → Secrets and variables → Actions → Variables**

| Variable | Value |
|----------|-------|
| `GITOPS_REPO` | `YOUR_GITHUB_USERNAME/zen-gitops` |

**Settings → Environments** — create two environments:

| Environment | Protection rule |
|-------------|----------------|
| `dev` | None |
| `prod` | Required reviewers |

### zen-pharma-frontend

**Settings → Secrets and variables → Actions → Secrets**

| Secret | Value |
|--------|-------|
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID |
| `GITOPS_TOKEN` | Same PAT as above |

**Settings → Secrets and variables → Actions → Variables**

| Variable | Value |
|----------|-------|
| `GITOPS_REPO` | `YOUR_GITHUB_USERNAME/zen-gitops` |

---

### How to get `NVD_API_KEY`

Java CI runs OWASP Dependency Check, which pulls vulnerability data from the NIST National Vulnerability Database. Without an API key, NVD rate-limits requests and the step runs slowly. The key is optional — CI still works without it.

1. Go to [https://nvd.nist.gov/developers/request-an-api-key](https://nvd.nist.gov/developers/request-an-api-key).
2. Fill in the form (name, organisation, email, intended use).
3. NIST sends a verification email — click the link inside it.
4. After verifying, NIST emails your API key.
5. In GitHub: **Settings → Secrets and variables → Actions → New repository secret**.
   - **Name:** `NVD_API_KEY`
   - **Value:** paste the key from NIST.
6. Save. Workflows that call `_java-pr-check.yml` / `_java-build.yml` already use `secrets: inherit`, so no further changes are needed.

---

## Step 4 — Trigger a CI pipeline

Push a commit to the `develop` branch of `zen-pharma-backend` (or `zen-pharma-frontend`) to trigger the full CI run.

```bash
git checkout develop
git commit --allow-empty -m "chore: trigger CI"
git push origin develop
```

Or go to **Actions → select any `ci-*.yml` workflow → Run workflow** on `develop`.

**What the pipeline does:**
1. Lint + test + build
2. Build Docker image
3. Push image to ECR with tag = commit SHA
4. Update `envs/dev/values-<service>.yaml` in `zen-gitops` with the new image tag
5. ArgoCD detects the change and rolls out automatically

---

## Step 5 — Validate images in ECR

1. Go to **AWS Console → ECR → Repositories**.
2. Open the repository for the service you triggered (e.g. `auth-service`).
3. Confirm a new image tag matching the commit SHA appears.

Or via CLI:

```bash
aws ecr describe-images \
  --repository-name auth-service \
  --region YOUR_REGION \
  --query 'sort_by(imageDetails, &imagePushedAt)[-1]'
```

---

## Step 6 — Validate ArgoCD sync

1. Get the ArgoCD URL:
   ```bash
   kubectl get svc -n argocd argocd-server
   ```
2. Log in (default admin password):
   ```bash
   kubectl -n argocd get secret argocd-initial-admin-secret \
     -o jsonpath="{.data.password}" | base64 -d
   ```
3. Open the ArgoCD UI → confirm the application for the deployed service shows:
   - **Status:** `Synced`
   - **Health:** `Healthy`
4. Confirm the pod is running with the new image:
   ```bash
   kubectl get pods -n pharma-dev
   kubectl describe pod <pod-name> -n pharma-dev | grep Image:
   ```

> If the app shows `OutOfSync`, click **Sync** manually or wait for the auto-sync interval (default 3 min).

---

## Reference

| Resource | Description |
|----------|-------------|
| [`CI-ARCHITECTURE.md`](./CI-ARCHITECTURE.md) | Full CI/CD architecture, OIDC, ECR policy, gitops layout |
| [`POST-DEPLOYMENT-WORKFLOW.md`](./POST-DEPLOYMENT-WORKFLOW.md) | QA and PROD promotion steps |
| [`.github/workflows/promote-prod.yml`](./.github/workflows/promote-prod.yml) | Manual PROD promotion workflow |
